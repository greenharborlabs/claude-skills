---
name: greenharbor-code-review
description: |
  Universal code review with multiple modes: diff-based (default), full-project,
  scoped (package/service/directory), and endpoint-flow tracing. Detects tech stack,
  applies specialized review checklists, merges and deduplicates findings. Supports
  Java/Spring and React/TS stacks. Use when user wants a deep code review, says
  "greenharbor-code-review", wants a pre-landing check, or needs production-readiness assessment.
  Modes: /greenharbor-code-review (diff), /greenharbor-code-review --scope src/billing/ (scoped),
  /greenharbor-code-review --full (full project), /greenharbor-code-review --flow "POST /api/orders" (endpoint trace).
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
  /greenharbor-code-review --baseline HEAD~5                  Diff against last 5 commits
  /greenharbor-code-review --scope src/billing/               Review a specific directory
  /greenharbor-code-review --scope OrderService               Review files matching a name
  /greenharbor-code-review --scope "src/billing/**/*.java"    Review files matching a glob
  /greenharbor-code-review --full                             Full project review (decomposed)
  /greenharbor-code-review --flow "POST /api/orders"          Trace and review an endpoint flow

Flags:
  --baseline <ref>       Override diff base (default: origin/main)
  --branch <name>        Alias for --baseline
  --critical-only        Only report CRITICAL findings
  --fix                  Auto-apply fixes for CRITICAL IMPL issues
  --interactive          Prompt on CRITICAL findings after reporting
  --out <path>           Write report to file

Output:
  Findings are classified into two resolution paths:
  - DESIGN items -> need /greenharbor-plan-work before implementation
  - IMPL items   -> surgical fixes, ready for /greenharbor-orchestrate
  CRITICAL IMPL findings are also written to an orchestrate-ready mini-plan file.
```

---

## Step 1: Parse arguments

- **Mode** (mutually exclusive, first match):
  - `--full` -> FULL mode
  - `--scope <target>` -> SCOPE mode
  - `--flow "<method> <path>"` -> FLOW mode
  - _(default)_ -> DIFF mode

- **Flags**:
  - `--baseline <ref>` or `--branch <ref>` -> custom diff base (default: `origin/main`)
  - `--critical-only` -> suppress INFORMATIONAL findings
  - `--fix` -> auto-apply CRITICAL IMPL fixes
  - `--interactive` -> prompt on CRITICAL findings after reporting
  - `--out <path>` -> write report to file. Auto-generated for non-DIFF modes.

---

## Step 2: Token-aware intake

Before reading code or diffs, size the review and choose the smallest sufficient input.

1. Start with metadata: `git diff --stat`, `git diff --numstat`, and `git diff --name-only` for DIFF mode; file counts and line counts for SCOPE/FULL/FLOW.
2. Exclude generated, binary, vendored, build, lock, snapshot, and minified files unless they are the explicit target.
3. Prefer patches over whole files in DIFF mode. Read whole files only when the patch lacks enough context to verify a finding.
4. If input is large, batch by file/module and synthesize findings after all batches:
   - DIFF: max 20 changed files or 1,200 patch lines per batch.
   - SCOPE/FLOW: max 20 source files or 4,000 source lines per batch.
   - FULL: max 15 files or 3,000 source lines per section.
5. Keep batch notes terse: confirmed findings, false-positive suppressions, and unresolved context needed. Do not carry raw code between batches.
6. If limits are exceeded, report the batching plan before reviewing and continue section by section.

---

## Step 3: Resolve review scope

### DIFF mode (default)

1. Determine baseline ref (default: `origin/main`, or `--baseline` value).
2. Get current branch: `git branch --show-current`
3. If on `main` with no changes, output "Nothing to review" and stop.
4. Run `git fetch origin --quiet` when `origin` exists.
5. Run `git diff <baseline> --stat`, `git diff <baseline> --numstat`, and `git diff <baseline> --name-only`.
6. Read patch content in token-aware batches using `git diff <baseline> -- <file...>`. Use `--unified=80` only for files needing broader context.

### SCOPE mode

For each target (comma-separated or space-separated):

1. **Resolve to files:**
   - Directory path -> search for source files under it
   - Glob pattern -> search directly
   - Name -> search for class/interface/function definitions and matching filenames
2. **Derive scope label** — human-readable name for report headers.
3. If no files found, warn and continue.
4. **Find direct dependencies** — parse import/require statements, add local imports as read-only context (cap at 15 files or 2,000 lines).
5. Read resolved files in token-aware batches. If a target resolves to more than 40 source files, decompose it like FULL mode and label each section.

### FULL mode

1. Build project file tree, excluding: `node_modules/`, `.git/`, `build/`, `target/`, `dist/`, `*.lock`
2. **Decompose into sections** (max 15 files or 3,000 source lines per section):
   - Group by package/module boundaries, then architectural layer
   - Identify cross-cutting concerns as their own section
3. Record per section: id, name, files, focus area, priority (high/medium/low)
4. Order foundational layers first.

### FLOW mode

1. Parse HTTP method and path from argument.
2. **Find the controller/route handler:**
   - Java/Spring: search for `@PostMapping`, `@GetMapping` etc. with the path pattern
   - React/Express/Node: search for `router.post`, route definitions
3. **Trace the call chain:** Controller -> Service(s) -> Repository/Client(s) -> Entities/DTOs -> Config
4. Read files in call-chain order, capped at 20 source files or 4,000 lines per batch. If the chain exceeds the cap, review controller/service/repository layers separately and synthesize.

---

## Step 4: Detect tech stack

Using the file list, classify:

### Stack detection
- **`java`**: `*.java`, `*.gradle`, `pom.xml`, or `*.properties`/`*.yml` under `src/`
- **`react`**: `*.tsx`, `*.ts`, `*.jsx` with React patterns (imports, components, hooks)

### Flag detection
- **`security`**: files containing `Security`, `Auth`, `Filter`, `Token`, `Credential`, `@PreAuthorize`, `SecurityFilterChain`, `jwt`, `oauth`
- **`performance`**: `@Repository`, `@Cacheable`, `RestTemplate`, `WebClient`, `@Transactional`, `@Query`, `@Async`, `cache`
- **`concurrency`**: `synchronized`, `ReentrantLock`, `Atomic*`, `CompletableFuture`, `@Async`, `@Scheduled`, `ExecutorService`, `ConcurrentHashMap`, `volatile`, `virtual.enabled`
- **`api-contract`**: `@RestController`, `@*Mapping`, `@RequestBody`, `@PathVariable`, `@ControllerAdvice`, `@ExceptionHandler`, `Pageable`, `@JsonProperty`

---

## Step 5: Load checklists

Always load `./references/checklist-universal.md`.

Then load stack-specific checklists:
- If `java` detected -> also load `./references/checklist-java-spring.md`
- If `react` detected -> also load `./references/checklist-react-ts.md`

**If `checklist-universal.md` cannot be loaded, STOP and report the error.**

---

## Step 6: Conduct reviews

Based on mode and detected stacks, perform specialized reviews.

### Review prompt structure (all modes)

For each review pass, trace through the code checking:
- Every constructor parameter -> is the field actually read/called? Dead weight if not.
- Every injected dependency -> is it used? Could it be derived from another?
- Every field with a setter -> is the field read anywhere? Dead state if not.

**Output format for all findings:**
```
[file:line] SEVERITY Problem description
Fix: suggested fix
Files: <comma-separated list of files the fix would touch>
```
Where SEVERITY is: CRITICAL, HIGH, MEDIUM, LOW

Skip preamble, summaries, and "looks good" comments. Only flag real problems.

### DIFF mode reviews

Review the diff content through these lenses based on detected flags:
- **General review** — always run
- **Security lens** — if `security` flag detected
- **Performance lens** — if `performance` flag detected
- **Concurrency lens** — if `concurrency` flag detected
- **API contract lens** — if `api-contract` flag detected

### SCOPE mode reviews

Same flag-based lenses as DIFF mode, applied to the resolved file contents.
Include context files as read-only background (do NOT review context files).

### FULL mode reviews

Review each section independently, applying all relevant lenses.
After all sections, perform a cross-cutting synthesis:
1. **Inconsistencies:** Patterns used in one section but not another?
2. **Dependency risk:** Most-referenced components lacking defensive coding?
3. **Missing integration:** Boundaries between sections with no test coverage?
4. **Architectural concerns:** Systemic issues visible only across sections?

### FLOW mode reviews

Review the call chain end-to-end:
1. **Authorization:** Is the endpoint secured? Can users access others' data?
2. **Input validation:** Validated at controller? Injection risks?
3. **Transaction boundaries:** Scoped correctly? Partial commit risk?
4. **Error handling:** What happens when each layer fails?
5. **Performance:** N+1 queries? Missing pagination? Unbounded results?
6. **Data consistency:** Race conditions? Check-then-act without atomicity?
7. **Response leakage:** Internal IDs, stack traces, or sensitive fields exposed?

Also apply flag-based lenses (security, performance, etc.) to the flow files.

---

## Step 7: Checklist review

Apply the loaded checklists against the review input:

**Pass 1 (CRITICAL):** Apply all CRITICAL checks from every checklist.
**Pass 2 (INFORMATIONAL):** Apply all INFORMATIONAL checks. (Skip if `--critical-only`.)

Follow the suppressions from each checklist — do NOT flag suppressed items.

For FULL mode, apply checklists per-section (not all at once) to keep scope bounded.

---

## Step 8: Merge, classify, and output

### 8a. Deduplicate and map severity

1. **Deduplicate:** If multiple passes flag the same issue at the same location, keep the more specific one.
2. **Map severity:**
   - CRITICAL or HIGH -> CRITICAL
   - MEDIUM or LOW -> INFORMATIONAL
3. If `--critical-only`, drop all INFORMATIONAL findings.

### 8b. Classify each finding by resolution path

**DESIGN** (`/greenharbor-plan-work`) if ANY apply:
- Requires new abstraction, pattern, or architectural layer
- Cross-cutting (affects multiple files across different packages)
- Fix involves competing approaches with trade-offs
- Missing capability needing requirements thinking
- Requires schema migration or API contract change
- Scope unclear — problem described but solution needs exploration
- Involves redesigning a flow or restructuring responsibilities

**IMPL** (`/greenharbor-orchestrate`) if ALL apply:
- Fix is well-defined and surgical
- Contained to 1-3 files with no architectural decisions
- Examples: add `@Valid`, fix N+1 with `JOIN FETCH`, add null check, fix SQL injection

When in doubt, classify as DESIGN.

### 8c. Group findings into parallel waves

1. Determine each finding's **file footprint** (files the fix touches).
2. Build waves greedily (DESIGN and IMPL independently):
   - Findings whose file footprints don't overlap go in the same wave.
   - Cross-cutting/unbounded findings get their own solo wave.
3. Number waves sequentially: DESIGN waves first, then IMPL waves.
4. **Consolidate within waves:** Group related findings into single actionable workloads.

### 8d. Build the report

**Report header by mode:**
- DIFF: `Diff Review (<baseline>): N issues (X critical, Y informational)`
- SCOPE: `Scope Review [<scope-label>]: N issues (...)`
- FULL: `Full Project Review: N issues (...) across M sections`
- FLOW: `Flow Review (<method> <path>): N issues (...)`

**Report structure:**

```
<Report Header>

## DESIGN — needs `/greenharbor-plan-work`
### CRITICAL
- **D1.** [file:line] Problem
  Why design needed: <one line>
  Files: <list>
### INFORMATIONAL
- **D2.** ...

## IMPL — ready for `/greenharbor-orchestrate`
### CRITICAL
- **I1.** [file:line] Problem
  Fix: <specific fix>
  Files: <list>
### INFORMATIONAL
- **I2.** ...

## Summary
| Category | Critical | Informational | Total |
|---|---:|---:|---:|
| DESIGN | X | X | X |
| IMPL | X | X | X |
| Total | X | X | X |

## Action Playbook
Group commands by parallel-safe wave. Use `/greenharbor-plan-work "<description>" --review {REPORT_PATH}` for DESIGN and `/greenharbor-orchestrate {IMPL_PLAN_PATH} --scope "Wave N"` for IMPL.
```

**SCOPE mode additions:** Prefix findings with scope label. Group by scope label for multi-target.

**FULL mode additions:** Use section grouping. Include cross-cutting findings separately.

**FLOW mode additions:** Add flow chain summary at top:
```
Flow chain: Controller -> Service -> Repository -> Entity -> Config

Structural follow-up: For class-design analysis, run:
/greenharbor-code-review --scope <ServiceClass> for each service in the chain.
```

If no issues: `<Report Header>: No issues found.`

### 8e. Write report

**Always write to file** in SCOPE, FULL, and FLOW modes. Use `--out` path if specified,
otherwise auto-generate at `reports/greenharbor-code-review-<mode>-<YYYY-MM-DD>.md`.
In DIFF mode, write to file when `--out` is specified or when an IMPL mini-plan will be generated.
If DIFF mode needs an implicit report path, use `reports/greenharbor-code-review-diff-<YYYY-MM-DD>.md`.
Output to stdout regardless of mode.

### 8f. Write IMPL mini-plan

By default, generate an IMPL mini-plan only when CRITICAL IMPL findings exist.
Generate a mini-plan for INFORMATIONAL-only IMPL findings only if `--out` is specified.

Generate an orchestrate-ready plan file restructuring IMPL findings into
`## Wave N:` / `### W{N}-{NN}:` format:

```markdown
# IMPL Fixes from Review

**Source:** `{REPORT_PATH}`
**Generated:** <date>

## Summary
Surgical fixes identified by /greenharbor-code-review.

## Wave 1: <wave theme>

### W1-01: <finding title — imperative form>
<problem description>

**Files:** <files from finding>
**Acceptance criteria:**
- <fix restated as checkable condition>
- Existing tests continue to pass
**Error handling:** N/A (surgical fix)
**Tests:** Verify via existing tests; add regression test if no coverage

### W1-02: <next finding>
...
```

Output path: `reports/greenharbor-code-review-impl-plan-<YYYY-MM-DD>.md`

**Branch propagation:** If `--branch`/`--baseline` was non-default, include in header
and append `--branch <ref>` to all playbook commands.

---

## Step 9: Handle critical issues (DIFF mode only)

Skip this step in SCOPE, FULL, and FLOW modes.

**If `--fix` flag set:**
- Auto-apply CRITICAL **IMPL** fixes only. Never auto-fix DESIGN items.
- Output summary of changes. Do NOT commit or push.

**If `--interactive` flag set and `--fix` not set:**
- For each CRITICAL **IMPL** finding, ask: Fix now? Acknowledge? False positive?
- For each CRITICAL **DESIGN** finding, ask: Acknowledge for plan-work? Acknowledge? False positive?
- Apply chosen fixes. Output ready-to-run `/greenharbor-plan-work` commands for DESIGN items.

**Default:**
- Report-only. Do not prompt or modify files.

---

## Important Rules

- **Read the FULL input before commenting.** Don't flag issues already addressed.
- **Read-only by default.** Only modify files if `--fix` is set or the user chooses "Fix now" in `--interactive` mode.
- **Stay token-aware.** Prefer metadata, targeted patches, batching, and terse notes over loading whole projects or full diffs.
- **Be terse.** One line problem, one line fix.
- **Only flag real problems.** Skip anything that's fine.
- **Respect scope.** Only flag issues in resolved files, not context files.
