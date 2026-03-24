---
name: review
version: 3.0.0
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
---

# Code Review

You are running the `/review` workflow — a multi-mode code review tool.

---

## Help

If the argument is `--help` or `help`, print this usage summary and stop:

```
/review — Universal code review with actionable triage

Usage:
  /review                                    Diff against main (default)
  /review --baseline release/2.1             Diff against a specific ref
  /review --baseline HEAD~5                  Diff against last 5 commits
  /review --scope src/billing/               Review a specific directory
  /review --scope OrderService               Review files matching a name
  /review --scope "src/billing/**/*.java"    Review files matching a glob
  /review --full                             Full project review (decomposed)
  /review --flow "POST /api/orders"          Trace and review an endpoint flow

Flags:
  --baseline <ref>       Override diff base (default: origin/main)
  --critical-only        Only report CRITICAL findings
  --fix                  Auto-apply fixes for CRITICAL IMPL issues (skips interactive)
  --out <path>           Write report to file (auto-generated for non-diff modes)
  --plan                 (--full only) Also write decomposition plan files

Output:
  Findings are classified into two resolution paths:
  - DESIGN items → need /plan-work before implementation
  - IMPL items   → surgical fixes, ready for /orchestrate
  Each finding includes a copy-paste command to kick off the next workflow.

Examples:
  /review                                    Pre-landing diff review
  /review --baseline develop                 Diff against develop branch
  /review --scope src/order/                 Review the order package
  /review --scope l402-auth/                 Review a Maven/Gradle submodule
  /review --scope l402-auth/,l402-payment/   Review multiple submodules
  /review --scope PaymentService             Find and review PaymentService + deps
  /review --flow "GET /api/users/{id}"       Trace GET user endpoint end-to-end
  /review --full --critical-only --out reports/review.md
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
  - `--baseline <ref>` → custom diff base (default: `origin/main`). Only applies to DIFF mode.
  - `--critical-only` → suppress INFORMATIONAL findings in output
  - `--fix` → auto-apply CRITICAL fixes without interactive prompts
  - `--out <path>` → write final report to this file path. In SCOPE, FULL, and FLOW modes, if `--out` is not specified, auto-generate a report file at `reports/review-<mode>-<YYYY-MM-DD>.md` (create the `reports/` directory if needed). In DIFF mode, `--out` is optional — output goes to stdout only unless specified.
  - `--plan` → (FULL mode only) write decomposition plan files to `plans/review/`

---

## Step 2: Resolve review scope

Based on the mode, determine which files are in scope for review.

### DIFF mode (default)

1. Determine the baseline ref. Default is `origin/main`. If `--baseline` was provided, use that value instead.
2. Run `git branch --show-current` to get the current branch.
3. If on `main` and baseline is `origin/main` and there are no uncommitted changes, output: **"Nothing to review — you're on main with no changes."** and stop.
4. Run `git fetch origin --quiet` then `git diff <baseline> --stat` to check for a diff. If no diff, output **"No changes found against `<baseline>`."** and stop.
5. Run `git diff <baseline> --name-only` to get the **file list**.
6. Run `git diff <baseline>` to get the **full diff content**.
7. The **review input** is the diff content. The **file list** is used for stack detection.

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
   - Cap context files at 30 to avoid scope explosion
5. The **review input** is the full content of all resolved files (read them). The **context files** are provided to agents as read-only background.
6. The **file list** is the resolved files (not including context files) — used for stack detection.
7. If multiple targets were provided, group files by their scope label for the report output. Each scope label becomes a section heading in the report.

### FULL mode

1. Build the project file tree using `ls` and Glob to find all source files, excluding: `node_modules/`, `.git/`, `build/`, `target/`, `dist/`, `*.lock`, `*.min.*`
2. **Decompose into sections** following these rules:
   - Each section maps to a single logical concern (do not mix orchestration with persistence)
   - No section should contain more than 15-20 files — split further if needed
   - Group by: package/module boundaries first, then by architectural layer
   - Identify cross-cutting concerns (security, logging, error handling) as their own section
3. For each section, record:
   - **id**: Sequential (e.g., `section-001`)
   - **name**: Short descriptive name (e.g., "Order Service Layer", "Security Configuration")
   - **files**: List of file paths
   - **focus**: What the reviewer should pay attention to
   - **priority**: `high` (core business logic, security), `medium` (services, integrations), `low` (config, utilities)
4. Order sections so foundational layers are reviewed before dependent layers.
5. If `--plan` flag is set, write decomposition files to `plans/review/` (see Step 2a below).
6. The **review input** is the set of sections. Each section is reviewed independently by a sub-agent.
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
6. The **review input** is the full content of all files in the flow chain, presented in call-chain order.
7. The **file list** is all files in the chain — used for stack detection.

### Step 2a: Write plan files (FULL mode + `--plan` only)

If `--plan` was passed with `--full`, write decomposition plan files:

1. Create `plans/review/00-master-plan.md` with:
   - JSON block containing all sections (id, name, files, focus, priority)
   - Recommended review order with rationale
2. For each section, write `plans/review/NN-<section-name>.md` with:
   - Section metadata
   - Files in scope
   - Review objectives
   - Ready-to-use sub-agent prompt

---

## Step 3: Detect tech stack

Using the **file list** from Step 2, classify into stacks and flags:

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

## Step 4: Read checklists

Always read `~/.claude/skills/review/checklist-universal.md`.

Then read stack-specific checklists based on detection:
- If `java` detected → also read `~/.claude/skills/review/checklist-java-spring.md`
- If `react` detected → also read `~/.claude/skills/review/checklist-react-ts.md`

Additionally, Glob for `~/.claude/skills/review/checklist-*.md` and load any extra checklists not already loaded. This allows user-added checklists (e.g., `checklist-python.md`, `checklist-go.md`) to be picked up automatically.

**If `checklist-universal.md` cannot be read, STOP and report the error.** Stack-specific checklists are optional — warn if missing but continue.

---

## Step 5: Dispatch reviewer agents

Based on mode and detected stacks, launch specialized reviewer agents **in parallel** using the Agent tool.

### DIFF mode — agent dispatch

Same as v2:
- `java` detected → launch `backend-reviewer-java`
- `java` + `security` flag → also launch `backend-security-reviewer-java`
- `java` + `performance` flag → also launch `backend-performance-reviewer-java`
- `java` + `concurrency` flag → also launch `backend-concurrency-reviewer-java`
- `java` + `api-contract` flag → also launch `backend-api-contract-reviewer-java`
- `react` detected → launch `frontend-reviewer`

**Agent prompt for DIFF mode:**
```
Review this diff for production-readiness issues. Be terse — output ONLY findings in this exact format:

[file:line] SEVERITY Problem description
Fix: suggested fix

Where SEVERITY is one of: CRITICAL, HIGH, MEDIUM, LOW

Skip preamble, summaries, and "looks good" comments. Only flag real problems.

Diff:
<full diff content>
```

### SCOPE mode — agent dispatch

Determine the number of agents based on file count:
- 1-10 files → 1 agent (matching the primary stack)
- 11-25 files → 2 agents (primary stack reviewer + security or performance if flagged)
- 26+ files → 3 agents (split files across agents by concern)

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

Where SEVERITY is one of: CRITICAL, HIGH, MEDIUM, LOW

Context files (read-only, for understanding dependencies — do NOT review these):
<list context file paths>

Files to review:
<for each file, include its full content with file path header>
```

### FULL mode — agent dispatch

For each section from Step 2, launch a reviewer agent **in parallel** (up to 6 concurrent agents; queue the rest):

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

Where SEVERITY is one of: CRITICAL, HIGH, MEDIUM, LOW

Skip preamble, summaries, and "looks good" comments. Only flag real problems.

Files to review:
<for each file in section, include its full content with file path header>
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

Launch a single deep-review agent with the full call chain:

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

Where SEVERITY is one of: CRITICAL, HIGH, MEDIUM, LOW

Flow files (in call-chain order):
<for each file in the chain, include its full content with file path header and layer label>
```

If `security` flag is detected, also launch `backend-security-reviewer-java` or `frontend-reviewer` (as appropriate) with the same flow files.

---

## Step 6: Checklist review

This runs **in parallel** with agent dispatch, regardless of mode.

Apply the loaded checklists against the **review input** (diff content, file content, or section content depending on mode):

1. **Pass 1 (CRITICAL):** Apply all CRITICAL checks from every loaded checklist.
2. **Pass 2 (INFORMATIONAL):** Apply all INFORMATIONAL checks from every loaded checklist.

If `--critical-only` flag is set, skip Pass 2 entirely.

Follow the suppressions from each checklist — do NOT flag suppressed items.

---

## Step 7: Merge, classify, and output

After all agents complete and checklist review finishes:

### 7a. Deduplicate and map severity

1. **Deduplicate**: If an agent finding and a checklist finding refer to the same issue at the same location, keep the more specific one. If two agents flag the same issue, merge into one finding.
2. **Map severity**: Agent findings map as follows:
   - Agent `CRITICAL` or `HIGH` → `CRITICAL`
   - Agent `MEDIUM` or `LOW` → `INFORMATIONAL`
3. If `--critical-only`, drop all INFORMATIONAL findings.

### 7b. Classify each finding by resolution path

For every finding, determine the appropriate resolution path. This is the most important part of the report — it tells the user exactly which workflow to kick off next.

**Classification rules:**

A finding is **DESIGN** (`/plan-work`) if ANY of these apply:
- It requires introducing a new abstraction, pattern, or architectural layer
- It affects multiple files across different packages/modules (cross-cutting)
- The fix involves a choice between competing approaches with trade-offs
- It's a missing capability that needs requirements thinking (e.g., "no rate limiting", "no retry strategy", "no auth on this endpoint group")
- It requires a schema migration or API contract change
- The scope of the fix is unclear — you can describe the problem but the solution needs exploration
- It involves redesigning an existing flow or restructuring responsibilities between classes

A finding is **IMPL** (`/orchestrate`) if ALL of these apply:
- The fix is well-defined and surgical — you know exactly what to change
- It's contained to 1-3 files with no architectural decisions needed
- Examples: add a missing `@Valid` annotation, fix an N+1 with a `JOIN FETCH`, add a null check, add a missing `AbortController` cleanup, fix a SQL injection by switching to parameterized query, add `readOnly = true` to a `@Transactional`

When in doubt, classify as DESIGN. It's better to plan first than to start coding without understanding the full picture.

### 7c. Build the report

**Build the report header** based on mode:
- DIFF mode: `Diff Review (<baseline>): N issues (X critical, Y informational)`
- SCOPE mode (single target): `Scope Review [<scope-label>]: N issues (X critical, Y informational)`
- SCOPE mode (multi target): `Scope Review [<label-1>, <label-2>, ...]: N issues (X critical, Y informational)`
- FULL mode: `Full Project Review: N issues (X critical, Y informational) across M sections`
- FLOW mode: `Flow Review (<method> <path>): N issues (X critical, Y informational)`

**Output in the following format.** Findings are grouped first by resolution path (DESIGN vs IMPL), then by scope label (in SCOPE mode with multiple targets or FULL mode), then by severity. Each finding includes a one-line suggested command the user can copy-paste to kick off the work:

```
<Report Header>

---

## DESIGN — needs `/plan-work` before implementation

These items need architectural thinking, trade-off analysis, or multi-file design before code is written.

### CRITICAL

- **D1.** [file:line] Problem description
  Why design needed: <one line explaining why this can't be fixed surgically>
  Suggested: `/plan-work --spec "Design <brief description>"`

- **D2.** [file:line] Problem description
  Why design needed: <one line>
  Suggested: `/plan-work --spec "Design <brief description>"`

### INFORMATIONAL

- **D3.** [file:line] Problem description
  Why design needed: <one line>
  Suggested: `/plan-work --spec "Design <brief description>"`

---

## IMPL — ready for `/orchestrate`

These items are surgical, well-defined fixes that can go straight to implementation.

### CRITICAL

- **I1.** [file:line] Problem description
  Fix: <specific fix description>
  Suggested: `/orchestrate "<brief task description>"`

- **I2.** [file:line] Problem description
  Fix: <specific fix description>
  Suggested: `/orchestrate "<brief task description>"`

### INFORMATIONAL

- **I3.** [file:line] Problem description
  Fix: <specific fix description>
  Suggested: `/orchestrate "<brief task description>"`

---

## Summary

| Category | Critical | Informational | Total |
|----------|----------|---------------|-------|
| DESIGN   | X        | X             | X     |
| IMPL     | X        | X             | X     |
| **Total**| **X**    | **X**         | **X** |

### Action Playbook

Every finding as a copy-paste command, grouped by severity then ordered DESIGN before IMPL.

#### CRITICAL
```bash
# D1. <brief problem description>
/plan-work --spec "<Design description>"

# D2. <brief problem description>
/plan-work --spec "<Design description>"

# I1. <brief problem description>
/orchestrate "<task description>"

# I2. <brief problem description>
/orchestrate "<task description>"
```

#### INFORMATIONAL
```bash
# D3. <brief problem description>
/plan-work "<Design description>"

# I3. <brief problem description>
/orchestrate "<task description>"
```
```

For **SCOPE mode** (single target), prefix findings with the scope label:

```
Scope Review [l402-auth]: 5 issues (2 critical, 3 informational)

---

## DESIGN — needs `/plan-work` before implementation

### l402-auth

- **D1.** [l402-auth/src/.../MacaroonService.java:89] ...
  Why design needed: ...
  Suggested: `/plan-work --spec "Design ..."`

---

## IMPL — ready for `/orchestrate`

### l402-auth

- **I1.** [l402-auth/src/.../MacaroonCrypto.java:45] ...
  Fix: ...
  Suggested: `/orchestrate "..."`
```

For **SCOPE mode** (multiple targets), group findings by scope label within each resolution path:

```
Scope Review [l402-auth, l402-payment]: 8 issues (3 critical, 5 informational)

---

## DESIGN — needs `/plan-work` before implementation

### l402-auth
- **D1.** [l402-auth/src/.../MacaroonService.java:89] ...

### l402-payment
- **D2.** [l402-payment/src/.../InvoiceService.java:112] ...

---

## IMPL — ready for `/orchestrate`

### l402-auth
- **I1.** [l402-auth/src/.../MacaroonCrypto.java:45] ...

### l402-payment
- **I2.** [l402-payment/src/.../PaymentRepository.java:33] ...
```

For **FULL mode**, use section grouping within each resolution path:

```
## DESIGN — needs `/plan-work` before implementation

### Section: Security Configuration (priority: high)
- **D1.** [SecurityConfig.java:45] ...

### Section: Order Service Layer (priority: high)
- **D2.** [OrderService.java:112] ...

### Cross-Cutting
- **D3.** [multiple files] ...

---

## IMPL — ready for `/orchestrate`

### Section: Order Service Layer (priority: high)
- **I1.** [OrderRepository.java:33] ...

### Section: Payment Integration (priority: medium)
- **I2.** [PaymentClient.java:78] ...
```

For **FLOW mode**, add the flow chain summary at the top:

```
Flow Review (POST /api/orders): N issues

**Flow chain:** OrderController → OrderService → OrderRepository → Order (entity) → SecurityConfig

**Structural follow-up:** For class-design analysis (unused dependencies, constructor bloat) on files in this chain, run:
`/review --scope <ServiceClass>` for each service in the chain.

---

## DESIGN — needs `/plan-work` ...
## IMPL — ready for `/orchestrate` ...
```

If no issues found: `<Report Header>: No issues found.`

### 7d. Write report

**Always write to file** in SCOPE, FULL, and FLOW modes. Use `--out` path if specified, otherwise auto-generate at `reports/review-<mode>-<YYYY-MM-DD>.md` (for SCOPE mode with a single target, use `reports/review-scope-<scope-label>-<YYYY-MM-DD>.md` so the filename reflects the module). Create the `reports/` directory if needed.

In DIFF mode, only write to file if `--out` was explicitly specified.

Output to stdout regardless of mode.

---

## Step 8: Handle critical issues (DIFF mode only)

**This step only applies in DIFF mode.** In SCOPE, FULL, and FLOW modes the report is the deliverable — skip this step entirely. The user will address findings from the written report in a separate session.

If `--fix` flag is set:
- For each CRITICAL **IMPL** finding only, apply the recommended fix automatically.
- CRITICAL DESIGN findings are never auto-fixed — they need `/plan-work` first.
- After all fixes are applied, output a summary of what was changed.
- Do NOT commit, push, or create PRs.

If `--fix` is NOT set:
- For EACH critical **IMPL** finding, use a separate `AskUserQuestion` with:
  - The problem and recommended fix
  - Options: **A: Fix it now**, **B: Acknowledge**, **C: False positive — skip**
- For EACH critical **DESIGN** finding, use a separate `AskUserQuestion` with:
  - The problem and why it needs design
  - Options: **A: Start `/plan-work` now**, **B: Acknowledge**, **C: False positive — skip**
- After all critical questions are answered, output a summary of choices.
- If the user chose A on any IMPL issue, apply the recommended fixes.
- If the user chose A on any DESIGN issue, invoke the `/plan-work` skill with the suggested spec.

---

## Important Rules

- **Read the FULL input before commenting.** Do not flag issues already addressed in the code.
- **Read-only by default.** Only modify files if the user explicitly chooses "Fix it now" or passes `--fix`. Never commit, push, or create PRs.
- **Be terse.** One line problem, one line fix. No preamble.
- **Only flag real problems.** Skip anything that's fine.
- **Respect scope.** In SCOPE mode, only flag issues in the resolved files, not in context files. In FLOW mode, only flag issues relevant to the traced flow.
- **Agent failures are not fatal.** If an agent fails or times out, continue with checklist review results. Note the failure in the output.
