---
name: greenharbor-code-review
version: 3.3.0
description: |
  Universal code review with multiple modes: diff-based (default), full-project,
  scoped (package/service/directory), and endpoint-flow tracing. Detects tech stack,
  dispatches specialized reviewer agents in parallel, applies checklist review,
  merges and deduplicates findings. Supports Java/Spring and React/TS stacks.
allowed-tools:
  - Bash
  - Read
  - Edit
  - Write
  - Grep
  - Glob
  - Agent
  - AskUserQuestion
  - TaskCreate
  - TaskUpdate
---

# Code Review

You are running the `/greenharbor-code-review` workflow — a multi-mode code review tool.

---

## Help

If the argument is `--help` or `help`, print this usage summary and stop:

```
/greenharbor-code-review — Universal code review with actionable triage

Usage:
  /greenharbor-code-review                                    Diff against main (default)
  /greenharbor-code-review --branch develop                   Diff against a specific branch
  /greenharbor-code-review --baseline release/2.1             Diff against a specific ref
  /greenharbor-code-review --baseline HEAD~5                  Diff against last 5 commits
  /greenharbor-code-review --scope src/billing/               Review a specific directory
  /greenharbor-code-review --scope OrderService               Review files matching a name
  /greenharbor-code-review --scope "src/billing/**/*.java"    Review files matching a glob
  /greenharbor-code-review --full                             Full project review (decomposed)
  /greenharbor-code-review --flow "POST /api/orders"          Trace and review an endpoint flow

Flags:
  --baseline <ref>       Override diff base (default: origin/main)
  --branch <name>        Alias for --baseline (e.g. --branch develop)
  --critical-only        Only report CRITICAL findings
  --fix                  Auto-apply fixes for CRITICAL IMPL issues (skips interactive)
  --interactive          Prompt on CRITICAL findings after reporting
  --out <path>           Write report to file (auto-generated for non-diff modes)
  --plan                 (--full only) Also write decomposition plan files

Output:
  Findings are classified into two resolution paths:
  - DESIGN items → need /greenharbor-plan-work before implementation
  - IMPL items   → surgical fixes, ready for /greenharbor-orchestrate
  CRITICAL IMPL findings are also written to an orchestrate-ready mini-plan file.
  Each finding includes a copy-paste command to kick off the next workflow.

Examples:
  /greenharbor-code-review                                    Pre-landing diff review
  /greenharbor-code-review --branch develop                   Diff against develop branch
  /greenharbor-code-review --baseline develop                 Same as --branch develop
  /greenharbor-code-review --scope src/order/                 Review the order package
  /greenharbor-code-review --scope l402-auth/                 Review a Maven/Gradle submodule
  /greenharbor-code-review --scope l402-auth/,l402-payment/   Review multiple submodules
  /greenharbor-code-review --scope PaymentService             Find and review PaymentService + deps
  /greenharbor-code-review --flow "GET /api/users/{id}"       Trace GET user endpoint end-to-end
  /greenharbor-code-review --full --critical-only --out reports/greenharbor-code-review.md
```

---

## Step 1: Parse arguments

Parse the invocation arguments:

- **Mode** (mutually exclusive — first match wins):
  - `--full` → FULL mode
  - `--scope <target>` → SCOPE mode (target is the next argument)
  - `--flow "<method> <path>"` → FLOW mode (quoted string with HTTP method and path)
  - _(default)_ → DIFF mode

- **Flags**:
  - `--baseline <ref>` or `--branch <ref>` → custom diff base (default: `origin/main`). Only applies to DIFF mode. `--branch` is an alias for `--baseline`.
  - `--critical-only` → suppress INFORMATIONAL findings in output
  - `--fix` → auto-apply CRITICAL fixes without interactive prompts
  - `--interactive` → prompt on CRITICAL findings after reporting
  - `--out <path>` → write final report to this file path. In SCOPE, FULL, and FLOW modes, if `--out` is not specified, auto-generate a report file at `reports/greenharbor-code-review-<mode>-<YYYY-MM-DD>.md` (create the `reports/` directory if needed). In DIFF mode, `--out` is optional unless an IMPL mini-plan is generated.
  - `--plan` → (FULL mode only) write decomposition plan files to `plans/greenharbor-code-review/`

---

## Step 2: Token-aware intake

Before reading code, diffs, or launching agents, size the review and choose the smallest sufficient input.

1. Start with metadata: `git diff --stat`, `git diff --numstat`, and `git diff --name-only` for DIFF mode; file counts and line counts for SCOPE/FULL/FLOW.
2. Exclude generated, binary, vendored, build, lock, snapshot, and minified files unless they are the explicit target.
3. Prefer patches over whole files in DIFF mode. Read whole files only when the patch lacks enough context to verify a finding.
4. Batch large inputs before sending to agents:
   - DIFF: max 20 changed files or 1,200 patch lines per agent prompt.
   - SCOPE/FLOW: max 20 source files or 4,000 source lines per agent prompt.
   - FULL: max 15 files or 3,000 source lines per section.
5. Keep batch notes terse: confirmed findings, false-positive suppressions, and unresolved context needed. Do not carry raw code between batches.
6. If limits are exceeded, report the batching plan before review and continue section by section.

---

## Step 3: Resolve review scope

Based on the mode, determine which files are in scope for review.

### DIFF mode (default)

1. Determine the baseline ref. Default is `origin/main`. If `--baseline` was provided, use that value instead.
2. Run `git branch --show-current` to get the current branch.
3. If on `main` and baseline is `origin/main` and there are no uncommitted changes, output: **"Nothing to review — you're on main with no changes."** and stop.
4. Run `git fetch origin --quiet` when `origin` exists.
5. Run `git diff <baseline> --stat`, `git diff <baseline> --numstat`, and `git diff <baseline> --name-only`. If no diff, output **"No changes found against `<baseline>`."** and stop.
6. Read patch content in token-aware batches using `git diff <baseline> -- <file...>`. Use `--unified=80` only for files needing broader context.
7. The **review input** is the batched patch content. The **file list** is used for stack detection.

### SCOPE mode

`--scope` accepts one or more targets, space-separated or comma-separated (e.g., `--scope l402-auth/,l402-payment/` or `--scope l402-auth/ l402-payment/`).

For each target:

1. **Resolve the target to a set of files:**
   - If target is a directory path (ends with `/` or exists as a directory) → Glob for all source files under it: `<target>/**/*.{java,kt,ts,tsx,jsx,js,py,go,rs}`
   - If target is a glob pattern (contains `*` or `?`) → Glob directly with the pattern
   - If target is a name (no path separators, no glob chars) → Grep the codebase for files containing `class <target>`, `interface <target>`, `function <target>`, `export.*<target>`, or filenames matching `*<target>*`. Collect all matching files.

2. **Derive the scope label** — a human-readable name used in the report header, section headings, and output filename:
   - If target is a directory: use the directory name (e.g., `l402-auth/` → `l402-auth`, `src/main/java/com/example/billing/` → `billing`)
   - If target is a Maven/Gradle submodule (contains `pom.xml`, `build.gradle`, or `build.gradle.kts`): use the module artifact name from the build file if available, otherwise the directory name
   - If target is a class/interface name: use that name (e.g., `PaymentService`)
   - If target is a glob: use the most specific directory component (e.g., `src/billing/**/*.java` → `billing`)

3. If no files resolved for a target, output **"No files found matching scope `<target>`."** and continue to next target. If ALL targets resolve to zero files, stop.

After resolving all targets:

4. **Find direct dependencies** of all resolved files:
   - Parse import/require statements in the resolved files
   - For each import that points to a local file (not a library), add that file to a **context files** list
   - Cap context files at 15 files or 2,000 lines to avoid scope explosion
5. The **review input** is resolved files read in token-aware batches. If a target resolves to more than 40 source files, decompose it like FULL mode and label each section. The **context files** are provided to agents as read-only background.
6. The **file list** is the resolved files (not including context files) — used for stack detection.
7. If multiple targets were provided, group files by their scope label for the report output. Each scope label becomes a section heading in the report.

### FULL mode

1. Build the project file tree using `ls` and Glob to find all source files, excluding: `node_modules/`, `.git/`, `build/`, `target/`, `dist/`, `*.lock`, `*.min.*`
2. **Decompose into sections** following these rules:
   - Each section maps to a single logical concern (do not mix orchestration with persistence)
   - No section should contain more than 15 files or 3,000 source lines — split further if needed
   - Group by: package/module boundaries first, then by architectural layer
   - Identify cross-cutting concerns (security, logging, error handling) as their own section
3. For each section, record:
   - **id**: Sequential (e.g., `section-001`)
   - **name**: Short descriptive name (e.g., "Order Service Layer", "Security Configuration")
   - **files**: List of file paths
   - **focus**: What the reviewer should pay attention to
   - **priority**: `high` (core business logic, security), `medium` (services, integrations), `low` (config, utilities)
4. Order sections so foundational layers are reviewed before dependent layers.
5. If `--plan` flag is set, write decomposition files to `plans/greenharbor-code-review/` (see Step 3a below).
6. The **review input** is the set of bounded sections. Each section is reviewed independently by a sub-agent.
7. The **file list** is all source files — used for stack detection.

### FLOW mode

1. Parse the flow argument to extract HTTP method and path (e.g., `POST /api/orders`).
2. **Find the controller/route handler**:
   - For Java/Spring: Grep for `@PostMapping`, `@GetMapping`, `@PutMapping`, `@DeleteMapping`, `@RequestMapping` with the path pattern. Account for class-level `@RequestMapping` prefix.
   - For React/Express/Node: Grep for `router.post`, `app.post`, route definitions matching the path.
   - For other frameworks: Grep for the path string and HTTP method references nearby.
3. If no handler found, output **"Could not find a handler for `<method> <path>`."** and stop.
4. **Trace the call chain** from the handler:
   - Read the handler method/function
   - Identify service/usecase calls it makes → read those files
   - From services, identify repository/client/external calls → read those files
   - Collect DTOs/models/entities referenced in the chain
   - Collect configuration (security rules, transaction config) that governs the endpoint
5. Build the **flow chain** as an ordered list: Controller → Service(s) → Repository/Client(s) → Entities/DTOs → Config
6. The **review input** is files in the flow chain, presented in call-chain order and capped at 20 source files or 4,000 lines per batch. If the chain exceeds the cap, review controller/service/repository layers separately and synthesize.
7. The **file list** is all files in the chain — used for stack detection.

### Step 3a: Write plan files (FULL mode + `--plan` only)

If `--plan` was passed with `--full`, write decomposition plan files:

1. Create `plans/greenharbor-code-review/00-master-plan.md` with:
   - JSON block containing all sections (id, name, files, focus, priority)
   - Recommended review order with rationale
2. For each section, write `plans/greenharbor-code-review/NN-<section-name>.md` with:
   - Section metadata
   - Files in scope
   - Review objectives
   - Ready-to-use sub-agent prompt

---

## Step 4: Detect tech stack

Using the **file list** from Step 3, classify into stacks and flags:

### Stack detection
- **`java`**: any file matching `*.java`, `*.gradle`, `pom.xml`, or `*.properties`/`*.yml` under a `src/` directory
- **`react`**: any file matching `*.tsx`, `*.ts`, `*.jsx` that contains React/component patterns (imports from `react`, component definitions, JSX, hooks)

### Flag detection
- **`security`** flag: any file path contains `Security`, `Auth`, `Filter`, `Token`, or `Credential` (case-insensitive), OR file content contains: `@PreAuthorize`, `SecurityFilterChain`, `HttpSecurity`, `jwt`, `oauth`
- **`performance`** flag: file content contains: `@Repository`, `@Cacheable`, `RestTemplate`, `WebClient`, `@Transactional`, `@Query`, `@Async`, `connection pool`, `cache`
- **`concurrency`** flag: file content contains: `synchronized`, `ReentrantLock`, `AtomicReference`, `AtomicInteger`, `AtomicLong`, `CompletableFuture`, `StructuredTaskScope`, `@Async`, `@Scheduled`, `ExecutorService`, `ThreadPoolTaskExecutor`, `ConcurrentHashMap`, `volatile`, `@Version`, `ShedLock`, `parallelStream`, `virtual.enabled`
- **`api-contract`** flag: file content contains: `@RestController`, `@GetMapping`, `@PostMapping`, `@PutMapping`, `@DeleteMapping`, `@PatchMapping`, `@RequestMapping`, `@RequestBody`, `@PathVariable`, `@RequestParam`, `@ControllerAdvice`, `@ExceptionHandler`, `@Schema`, `@Operation`, `Pageable`, `@JsonProperty`, `@JsonAlias`

Record which stacks and flags were detected. Multiple stacks can be active simultaneously.

---

## Step 5: Read checklists

Always read `~/.claude/skills/greenharbor-code-review/checklist-universal.md`.

Then read stack-specific checklists based on detection:
- If `java` detected → also read `~/.claude/skills/greenharbor-code-review/checklist-java-spring.md`
- If `react` detected → also read `~/.claude/skills/greenharbor-code-review/checklist-react-ts.md`

Additionally, Glob for `~/.claude/skills/greenharbor-code-review/checklist-*.md` and load any extra checklists not already loaded. This allows user-added checklists (e.g., `checklist-python.md`, `checklist-go.md`) to be picked up automatically.

**If `checklist-universal.md` cannot be read, STOP and report the error.** Stack-specific checklists are optional — warn if missing but continue.

---

## Step 6: Dispatch reviewer agents

Based on mode and detected stacks, launch specialized reviewer agents **in parallel** using the Agent tool.

### DIFF mode — agent dispatch

- `java` detected → launch `greenharbor-backend-reviewer-java` (includes performance and concurrency checks)
- `java` + `security` flag → also launch `greenharbor-backend-security-reviewer-java`
- `java` + `api-contract` flag → also launch `greenharbor-backend-api-contract-reviewer-java`
- `react` detected → launch `greenharbor-frontend-reviewer`

**Agent prompt for DIFF mode:**
```
Review this diff for production-readiness issues.

For each class in the diff, actively trace:
- Every constructor parameter → is the field actually read/called in any method? If not, it's dead weight.
- Every injected dependency → is it used? Could it be derived from another already-injected dependency?
- Every field with a setter → is the field read anywhere? A setter with no reader is dead state.

Be terse — output ONLY findings in this exact format:

[file:line] SEVERITY Problem description
Fix: suggested fix
Files: <comma-separated list of files the fix would touch>

Where SEVERITY is one of: CRITICAL, HIGH, MEDIUM, LOW

Skip preamble, summaries, and "looks good" comments. Only flag real problems.

Diff batch:
<bounded patch content for this batch>
```

### SCOPE mode — agent dispatch

Dispatch agents using the same flag-based pattern as DIFF mode:
- `java` detected → launch `greenharbor-backend-reviewer-java` (includes performance and concurrency checks)
- `java` + `security` flag → also launch `greenharbor-backend-security-reviewer-java`
- `java` + `api-contract` flag → also launch `greenharbor-backend-api-contract-reviewer-java`
- `react` detected → launch `greenharbor-frontend-reviewer`

Launch all agents **in parallel**.

**Agent prompt for SCOPE mode:**
```
Review the following source files for production-readiness issues. These files represent the <scope target> area of the codebase.

For each class, actively trace:
- Every constructor parameter → is the field actually read/called in any method? If not, it's dead weight.
- Every injected dependency → is it used? Could it be derived from another already-injected dependency?
- Every field with a setter → is the field read anywhere? A setter with no reader is dead state.

Be terse — output ONLY findings in this exact format:

[file:line] SEVERITY Problem description
Fix: suggested fix
Files: <comma-separated list of files the fix would touch>

Where SEVERITY is one of: CRITICAL, HIGH, MEDIUM, LOW

Skip preamble, summaries, and "looks good" comments. Only flag real problems.

Context files (read-only, for understanding dependencies — do NOT review these):
<list context file paths>

Files to review:
<bounded source batch with file path headers>
```

### FULL mode — agent dispatch

For each section from Step 3, launch a reviewer agent **in parallel** (up to 6 concurrent agents; queue the rest):

**Agent prompt for FULL mode (per section):**
```
You are reviewing the "<section name>" section of a <detected stack> project.

Focus area: <section focus>
Priority: <section priority>

For each class, actively trace:
- Every constructor parameter → is the field actually read/called in any method? If not, it's dead weight.
- Every injected dependency → is it used? Could it be derived from another already-injected dependency?
- Every field with a setter → is the field read anywhere? A setter with no reader is dead state.

Be terse — output ONLY findings in this exact format:

[file:line] SEVERITY Problem description
Fix: suggested fix
Files: <comma-separated list of files the fix would touch>

Where SEVERITY is one of: CRITICAL, HIGH, MEDIUM, LOW

Skip preamble, summaries, and "looks good" comments. Only flag real problems.

Files to review:
<bounded section source with file path headers>
```

After all section agents complete, launch one additional **synthesis agent**:
```
You have received findings from multiple focused review agents across these sections:
<list section names>

Full findings:
<all agent findings concatenated>

Perform a cross-cutting synthesis:
1. INCONSISTENCIES: Are there patterns used in one section but not another? (e.g., input validation in one service but not a sibling)
2. DEPENDENCY RISK: Which components appear most frequently as dependencies? Flag any that lack defensive coding.
3. MISSING INTEGRATION: Are there boundaries between sections with no integration test coverage?
4. ARCHITECTURAL CONCERNS: Any systemic issues visible only when looking across sections?

Output ONLY findings in the same format:
[file:line] SEVERITY Problem description
Fix: suggested fix
```

### FLOW mode — agent dispatch

Launch the detected stack's primary reviewer agent (`greenharbor-backend-reviewer-java` for Java, `greenharbor-frontend-reviewer` for React) with the full call chain. If no stack detected, use a `general-purpose` agent.

**Agent prompt for FLOW mode:**
```
You are reviewing the end-to-end flow for: <METHOD> <PATH>

The call chain is:
<ordered list: Controller → Service → Repository/Client → Entity/DTO → Config>

While tracing the flow, also note any constructor parameters or injected dependencies in the traced classes that are not used by any method in the flow chain — these may indicate dead dependencies.

Review this flow for:
1. Authorization: Is the endpoint properly secured? Can users access other users' data?
2. Input validation: Is input validated at the controller? Are there injection risks?
3. Transaction boundaries: Are transactions scoped correctly? Any risk of partial commits?
4. Error handling: What happens when each layer fails? Are errors propagated correctly?
5. Performance: N+1 queries? Missing pagination? Unbounded result sets?
6. Data consistency: Race conditions? Check-then-act without atomicity?
7. Response leakage: Does the response expose internal IDs, stack traces, or sensitive fields?

Be terse — output ONLY findings in this exact format:

[file:line] SEVERITY Problem description
Fix: suggested fix
Files: <comma-separated list of files the fix would touch>

Where SEVERITY is one of: CRITICAL, HIGH, MEDIUM, LOW

Flow files (in call-chain order):
<bounded flow source with file path headers and layer labels>
```

Additionally, launch specialized agents based on detected flags (same pattern as DIFF mode), providing the flow files as review input:
- `security` flag → launch `greenharbor-backend-security-reviewer-java` or `greenharbor-frontend-reviewer` (as appropriate)
- `api-contract` flag → launch `greenharbor-backend-api-contract-reviewer-java`

Launch all flag-based agents **in parallel** with the deep-review agent.

---

## Step 7: Checklist review

How checklist review runs depends on the mode:

### DIFF, SCOPE, and FLOW modes

Launch a **checklist review agent** in parallel with the Step 6 reviewer agents (include it in the same Agent tool-use turn). This ensures checklist review runs concurrently with specialized agent reviews.

**Agent prompt for checklist review:**
```
Apply the following review checklists against the provided code. Perform two passes:

Pass 1 (CRITICAL): Apply all CRITICAL checks from every checklist.
Pass 2 (INFORMATIONAL): Apply all INFORMATIONAL checks from every checklist.
<if --critical-only flag is set, include: "Skip Pass 2 entirely.">

Follow the suppressions from each checklist — do NOT flag suppressed items.

Be terse — output ONLY findings in this exact format:

[file:line] SEVERITY Problem description
Fix: suggested fix
Files: <comma-separated list of files the fix would touch>

Where SEVERITY is one of: CRITICAL, HIGH, MEDIUM, LOW

Skip preamble, summaries, and "looks good" comments. Only flag real problems.

Checklists:
<loaded checklist criteria>

Code to review:
<review input: diff content, file content, or section content depending on mode>
```

Use the `general-purpose` agent type (not a specialized reviewer) and set the model to `sonnet` for speed.

### FULL mode

Do NOT launch a standalone checklist agent — the review input (all project source files) is too large for a single agent. Instead, append the checklist criteria to each section agent's prompt from Step 6. Add this block to the end of each section agent prompt:

```
Additionally, apply the following review checklists against the files in this section:
<if --critical-only flag is set, include: "Apply only the CRITICAL checks. Skip INFORMATIONAL.">

<loaded checklist criteria>

Follow the suppressions from each checklist — do NOT flag suppressed items.
Include checklist findings in the same output format as above.
```

This distributes checklist review across section agents, keeping each agent's input bounded.

---

## Step 8: Merge, classify, and output

After all agents complete and checklist review finishes:

### 8a. Deduplicate and map severity

1. **Deduplicate**: If an agent finding and a checklist finding refer to the same issue at the same location, keep the more specific one. If two agents flag the same issue, merge into one finding.
2. **Map severity**: Agent findings map as follows:
   - Agent `CRITICAL` or `HIGH` → `CRITICAL`
   - Agent `MEDIUM` or `LOW` → `INFORMATIONAL`
3. If `--critical-only`, drop all INFORMATIONAL findings.

### 8b. Classify each finding by resolution path

For every finding, determine the appropriate resolution path. This is the most important part of the report — it tells the user exactly which workflow to kick off next.

**Classification rules:**

A finding is **DESIGN** (`/greenharbor-plan-work`) if ANY of these apply:
- It requires introducing a new abstraction, pattern, or architectural layer
- It affects multiple files across different packages/modules (cross-cutting)
- The fix involves a choice between competing approaches with trade-offs
- It's a missing capability that needs requirements thinking (e.g., "no rate limiting", "no retry strategy", "no auth on this endpoint group")
- It requires a schema migration or API contract change
- The scope of the fix is unclear — you can describe the problem but the solution needs exploration
- It involves redesigning an existing flow or restructuring responsibilities between classes

A finding is **IMPL** (`/greenharbor-orchestrate`) if ALL of these apply:
- The fix is well-defined and surgical — you know exactly what to change
- It's contained to 1-3 files with no architectural decisions needed
- Examples: add a missing `@Valid` annotation, fix an N+1 with a `JOIN FETCH`, add a null check, add a missing `AbortController` cleanup, fix a SQL injection by switching to parameterized query, add `readOnly = true` to a `@Transactional`

When in doubt, classify as DESIGN. It's better to plan first than to start coding without understanding the full picture.

### 8c. Group findings into parallel waves

Before building the report, group findings into **waves** — sets of tasks that can be safely worked on in parallel because they touch different files.

**Algorithm:**

1. For each finding, determine its **file footprint** — the set of files the fix will touch:
   - IMPL findings: the file(s) listed in the finding (typically 1-3 files)
   - DESIGN findings: the file(s) listed plus any files mentioned in the "Why design needed" rationale. If the scope is unclear or cross-cutting, mark the finding as `unbounded` — it cannot be parallelized with anything.

2. Build waves greedily within each resolution path (DESIGN and IMPL independently):
   - Start with Wave 1 as empty, with an empty "claimed files" set.
   - For each finding (ordered by severity: CRITICAL first, then INFORMATIONAL):
     - If the finding is `unbounded`, assign it to its own solo wave.
     - If NONE of the finding's footprint files overlap with the current wave's claimed files, add the finding to the current wave and add its files to claimed files.
     - Otherwise, start a new wave and add the finding there.
   - After all findings are assigned, compact: merge any waves whose claimed file sets don't overlap.

3. Number waves sequentially across the entire report: Wave 1, Wave 2, etc. DESIGN waves come first, then IMPL waves. Within a wave, order by severity (CRITICAL before INFORMATIONAL).

4. **Consolidate within waves:** Within each wave, merge findings that share a common theme or module into a single actionable workload. Do NOT emit one command per finding — group related IMPL findings into one `/greenharbor-orchestrate` command targeting a single wave in the IMPL mini-plan, and related DESIGN findings into one `/greenharbor-plan-work` command. A wave with 5 small IMPL fixes in the same service layer should become 1 greenharbor-orchestrate task, not 5 separate commands. Use your judgment: findings touching different subsystems within the same wave stay as separate commands.

**Important:** The wave grouping is an optimization hint, not a hard constraint. If a finding's file footprint is ambiguous (e.g., "multiple files" with no specifics), be conservative and put it in its own wave.

### 8d. Build the report

**Build the report header** based on mode:
- DIFF mode: `Diff Review (<baseline>): N issues (X critical, Y informational)`
- SCOPE mode (single target): `Scope Review [<scope-label>]: N issues (X critical, Y informational)`
- SCOPE mode (multi target): `Scope Review [<label-1>, <label-2>, ...]: N issues (X critical, Y informational)`
- FULL mode: `Full Project Review: N issues (X critical, Y informational) across M sections`
- FLOW mode: `Flow Review (<method> <path>): N issues (X critical, Y informational)`

**Output in the following format.** Findings are grouped first by resolution path (DESIGN vs IMPL), then by scope label or section when relevant, then by severity. Each finding includes a one-line suggested command the user can copy-paste to kick off the work:

```
<Report Header>

## DESIGN — needs `/greenharbor-plan-work`
### CRITICAL
- **D1.** [file:line] Problem
  Why design needed: <one line>
  Files: <list of files>
### INFORMATIONAL
- **D2.** [file:line] Problem
  Why design needed: <one line>
  Files: <list of files>

## IMPL — ready for `/greenharbor-orchestrate`
### CRITICAL
- **I1.** [file:line] Problem
  Fix: <specific fix>
  Files: <list of files>
### INFORMATIONAL
- **I2.** [file:line] Problem
  Fix: <specific fix>
  Files: <list of files>

## Summary
| Category | Critical | Informational | Total |
|---|---:|---:|---:|
| DESIGN | X | X | X |
| IMPL | X | X | X |
| Total | X | X | X |

## Action Playbook
Group commands by parallel-safe wave. Use `/greenharbor-plan-work "<description>" --review {REPORT_PATH}` for DESIGN and `/greenharbor-orchestrate {IMPL_PLAN_PATH} --scope "Wave N"` for IMPL.
```

Mode additions: SCOPE prefixes findings with the scope label; FULL prefixes findings with the section; FLOW adds `Flow chain: Controller → Service → Repository/Client → Entity/DTO → Config` before findings.

Branch propagation: if `--branch` or non-default `--baseline` was set, append `--branch <ref>` to playbook commands. Strip an `origin/` prefix when propagating.

If no issues found: `<Report Header>: No issues found.`

### 8e. Write report

**Always write to file** in SCOPE, FULL, and FLOW modes. Use `--out` path if specified, otherwise auto-generate at `reports/greenharbor-code-review-<mode>-<YYYY-MM-DD>.md` (for SCOPE mode with a single target, use `reports/greenharbor-code-review-scope-<scope-label>-<YYYY-MM-DD>.md` so the filename reflects the module). Create the `reports/` directory if needed.

In DIFF mode, write to file when `--out` is specified or when an IMPL mini-plan will be generated. If DIFF mode needs an implicit report path, use `reports/greenharbor-code-review-diff-<YYYY-MM-DD>.md`.

Output to stdout regardless of mode.

### 8f. Write IMPL mini-plan

By default, generate an IMPL mini-plan only when CRITICAL IMPL findings exist. Generate a mini-plan for INFORMATIONAL-only IMPL findings only if `--out` is specified. Skip this step if there are no qualifying IMPL findings.

When qualifying IMPL findings exist, generate an orchestrate-ready plan file that restructures the IMPL findings from the report into the `## Wave N:` / `### W{N}-{NN}:` format that greenharbor-orchestrate parses via Rule A. This separates concerns: the review report stays human-optimized (bullet format, scannable), while the mini-plan is machine-optimized for greenharbor-orchestrate.

**Output path:** `reports/greenharbor-code-review-impl-plan-<YYYY-MM-DD>.md` (or `reports/greenharbor-code-review-impl-plan-<scope-label>-<YYYY-MM-DD>.md` in SCOPE mode). Create the `reports/` directory if needed.

**Mini-plan structure:**

```markdown
# IMPL Fixes from Review

**Source:** `{REPORT_PATH}`
**Generated:** <date>

## Summary
Surgical fixes identified by /greenharbor-code-review. Each work unit is a self-contained fix
that can be implemented without architectural decisions.

## Wave 1: <wave theme from Action Playbook grouping>

### W1-01: <I{N} finding title — imperative form>
<problem description from the finding>

**Files:** <files from the finding's Files: field>
**Acceptance criteria:**
- <derived from the Fix: field — restate as a checkable condition>
- Existing tests continue to pass
**Error handling:** N/A (surgical fix)
**Tests:** Verify fix via existing test suite; add regression test if no coverage exists

### W1-02: <next finding in same wave>
...

## Wave 2: <next wave theme>

### W2-01: ...
```

**Mapping rules:**
- Each **wave** in the Action Playbook that contains `/greenharbor-orchestrate` commands becomes a `## Wave N:` in the mini-plan
- Each **IMPL finding** (I1, I2, etc.) within that wave becomes a `### W{N}-{NN}:` work unit
- Grouped findings (e.g., "I1+I2") become separate work units in the same wave (they're independent — that's why they were grouped)
- The finding's `Fix:` field becomes the acceptance criterion (restated as a checkable condition)
- The finding's `Files:` field becomes the `**Files:**` list
- Wave numbering in the mini-plan is sequential starting at 1 (independent of the report's wave numbering, which mixes DESIGN and IMPL)

**Branch propagation:** If `--branch` or `--baseline` was set to a non-default value, include it in the mini-plan header as metadata so the user can pass it when invoking orchestrate:
```markdown
**Branch:** `<ref>` (pass `--branch <ref>` to `/greenharbor-orchestrate`)
```

After writing, print: `IMPL plan written to: <path> (<N> work units across <M> waves)`

---

## Step 9: Handle critical issues (DIFF mode only)

**This step only applies in DIFF mode.** In SCOPE, FULL, and FLOW modes the report is the deliverable — skip this step entirely. The user will address findings from the written report in a separate session.

If `--fix` flag is set:
- For each CRITICAL **IMPL** finding only, apply the recommended fix automatically.
- CRITICAL DESIGN findings are never auto-fixed — they need `/greenharbor-plan-work` first.
- After all fixes are applied, output a summary of what was changed.
- Do NOT commit, push, or create PRs.

If `--interactive` is set and `--fix` is NOT set:
- For EACH critical **IMPL** finding, use a separate `AskUserQuestion` with:
  - The problem and recommended fix
  - Options: **A: Fix it now**, **B: Acknowledge**, **C: False positive — skip**
- For EACH critical **DESIGN** finding, use a separate `AskUserQuestion` with:
  - The problem and why it needs design
  - Options: **A: Acknowledge — I'll run `/greenharbor-plan-work` next**, **B: Acknowledge**, **C: False positive — skip**
- After all critical questions are answered, output a summary of choices.
- If the user chose A on any IMPL issue, apply the recommended fixes.
- If the user chose A on any DESIGN issue, output the ready-to-run `/greenharbor-plan-work` command (including `--review {REPORT_PATH}` if a report was written, and `--branch <ref>` if a non-default baseline was used) so the user can invoke it in their next prompt. Do NOT attempt to invoke `/greenharbor-plan-work` directly — the review skill should remain read-only.

Default behavior when neither `--fix` nor `--interactive` is set:
- Report-only. Do not prompt or modify files.

---

## Important Rules

- **Read the FULL input before commenting.** Do not flag issues already addressed in the code.
- **Read-only by default.** Only modify files if `--fix` is set or the user chooses "Fix it now" in `--interactive` mode. Never commit, push, or create PRs.
- **Stay token-aware.** Prefer metadata, targeted patches, batching, and terse notes over loading whole projects or full diffs.
- **Be terse.** One line problem, one line fix. No preamble.
- **Only flag real problems.** Skip anything that's fine.
- **Respect scope.** In SCOPE mode, only flag issues in the resolved files, not in context files. In FLOW mode, only flag issues relevant to the traced flow.
- **Agent failures are not fatal.** If an agent fails or times out, continue with checklist review results. Note the failure in the output.
