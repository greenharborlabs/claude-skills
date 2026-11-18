---
name: greenharbor-code-review
version: 3.4.0
description: |
  Universal code review with multiple modes: diff-based (default), full-project,
  scoped (package/service/directory), and endpoint-flow tracing. Detects tech stack,
  dispatches specialized reviewer agents in parallel, applies checklist review,
  verifies, merges, and deduplicates findings. Supports Java/Spring, Rust, Python,
  and React/TS stacks.
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

You are running the `/greenharbor-code-review` workflow — a multi-mode code review tool.
Protect the context window: this session handles metadata, dispatch, verification,
and the report. Reviewer agents read code and diffs themselves; never paste diffs,
whole files, or checklist contents into agent prompts.

---

## Help

If the argument is `--help` or `help`, print this usage summary and stop:

```
/greenharbor-code-review — Universal code review with actionable triage

Usage:
  /greenharbor-code-review                                    Diff against the default branch
  /greenharbor-code-review --branch develop                   Diff against a specific branch
  /greenharbor-code-review --baseline release/2.1             Diff against a specific ref
  /greenharbor-code-review --baseline HEAD~5                  Diff against last 5 commits
  /greenharbor-code-review --scope src/billing/               Review a specific directory
  /greenharbor-code-review --scope OrderService               Review files matching a name
  /greenharbor-code-review --scope "src/billing/**/*.java"    Review files matching a glob
  /greenharbor-code-review --scope l402-auth/,l402-payment/   Review multiple modules
  /greenharbor-code-review --full                             Full project review (decomposed)
  /greenharbor-code-review --flow "POST /api/orders"          Trace and review an endpoint flow

Flags:
  --baseline <ref>       Override diff base (default: origin's default branch)
  --branch <name>        Alias for --baseline
  --critical-only        Only report CRITICAL findings
  --fix                  (diff mode) Auto-apply verified CRITICAL IMPL fixes
  --interactive          (diff mode) Prompt on CRITICAL findings after reporting
  --out <path>           Write report to file (auto-generated for non-diff modes)
  --plan                 (--full only) Also write decomposition plan files

Output:
  Findings are classified into two resolution paths:
  - DESIGN items → need /greenharbor-plan-work before implementation
  - IMPL items   → surgical fixes, ready for /greenharbor-orchestrate
  CRITICAL IMPL findings are also written to an orchestrate-ready mini-plan file.
  The Action Playbook gives a copy-paste command for each parallel-safe wave.
```

---

## Step 1: Parse arguments

- **Mode** (mutually exclusive — first match wins):
  - `--full` → FULL mode
  - `--scope <target>` → SCOPE mode (one or more targets, comma- or space-separated)
  - `--flow "<method> <path>"` → FLOW mode
  - _(default)_ → DIFF mode
- **Flags**:
  - `--baseline <ref>` / `--branch <ref>` → custom diff base. DIFF mode only.
  - `--critical-only` → suppress INFORMATIONAL findings.
  - `--fix`, `--interactive` → DIFF mode only (Step 7).
  - `--out <path>` → report path. SCOPE/FULL/FLOW always write a report (Step 6f).
  - `--plan` → FULL mode only: write decomposition files to `plans/greenharbor-code-review/`.

---

## Step 2: Resolve review scope

In every mode, exclude generated, binary, vendored, build output, lock, snapshot,
and minified files (`node_modules/`, `.git/`, `build/`, `target/`, `dist/`,
`*.lock`, `*.min.*`) unless they are the explicit target. Size with metadata
(file and line counts) before dispatching anything.

### DIFF mode (default)

1. If an `origin` remote exists, run `git fetch origin --quiet`. If it fails
   (offline), continue with local refs and note it in the report.
2. Resolve the baseline: the `--baseline`/`--branch` value if given (if it does
   not resolve but `origin/<value>` does, use that); otherwise the target of
   `git symbolic-ref --short refs/remotes/origin/HEAD`; otherwise the first of
   `origin/main`, `origin/master`, `main`, `master` that exists.
3. Compute `BASE=$(git merge-base <baseline> HEAD)`. Always diff against `BASE`,
   never the baseline tip — diffing the tip shows commits that landed on the
   baseline after you branched as if you had reverted them.
4. Collect the change set:
   - `git diff BASE --numstat` (committed plus uncommitted tracked changes).
   - `git ls-files --others --exclude-standard` (untracked new files, which
     `git diff` omits). Review these as whole new files.
   If both are empty, output **"No changes found against `<baseline>`."** and stop.
5. Batch the file list: max 20 files or 1,200 changed lines per batch (from
   numstat; count untracked files by line count).

### SCOPE mode

For each target:

1. **Resolve the target to files:**
   - Existing file path → that file.
   - Directory (ends with `/` or exists as a directory) → Glob source files under it:
     `<target>/**/*.{java,kt,ts,tsx,jsx,js,py,go,rs}`.
   - Glob pattern (contains `*` or `?`) → Glob directly.
   - Name (no separators or glob chars) → Grep for `class <target>`,
     `interface <target>`, `struct <target>`, `trait <target>`, `def <target>`,
     `function <target>`, `export.*<target>`, plus filenames matching `*<target>*`.
2. **Derive the scope label** for headings and filenames: the directory name
   (`src/main/java/com/example/billing/` → `billing`); for a Maven/Gradle/Cargo
   module, the artifact/package name from its build file if available; the
   class name for name targets; the most specific directory for globs.
3. If a target resolves to no files, output **"No files found matching scope
   `<target>`."** and continue. If all targets resolve to zero files, stop.

Then:

4. **Context files:** parse import/require/use statements in the resolved files and
   collect local (non-library) imports as read-only context, capped at 15 files.
5. Batch: max 20 source files or 4,000 source lines per batch. If a target
   resolves to more than 40 source files, decompose it like FULL mode and label
   each section.
6. With multiple targets, group the report by scope label.

### FULL mode

1. Find all source files (Glob), applying the exclusions above.
2. **Decompose into sections:**
   - One logical concern per section (do not mix orchestration with persistence).
   - Max 15 files or 3,000 source lines per section — split further if needed.
   - Group by package/module boundaries first, then architectural layer.
   - Give cross-cutting concerns (security, logging, error handling) their own section.
3. Record per section: `id` (`section-001`…), `name`, `files`, `focus`, and
   `priority` (`high` core logic/security, `medium` services/integrations, `low`
   config/utilities). Order foundational layers before dependent layers.
4. If `--plan`: write `plans/greenharbor-code-review/00-master-plan.md` (JSON block
   of all sections plus recommended review order with rationale) and one
   `NN-<section-name>.md` per section (metadata, files, review objectives, a
   ready-to-use reviewer prompt).

### FLOW mode

1. Parse the HTTP method and path (e.g., `POST /api/orders`).
2. **Find the handler:**
   - Java/Spring: `@PostMapping`/`@GetMapping`/`@PutMapping`/`@DeleteMapping`/
     `@PatchMapping`/`@RequestMapping` with the path; account for class-level
     `@RequestMapping` prefixes.
   - Node/Express: `router.post`, `app.post`, and route definitions matching the path.
   - Python: FastAPI `@app.post`/`@router.post` (with router prefixes), Flask
     `@app.route(..., methods=[...])`/`@bp.post`.
   - Rust: Axum `.route("<path>", post(handler))`, Actix `#[post("<path>")]`.
   - Otherwise: Grep for the path string near HTTP method references.
3. If no handler is found, output **"Could not find a handler for `<method> <path>`."**
   and stop.
4. **Trace the call chain:** handler → service/use-case calls → repository/client/
   external calls → DTOs/models/entities → configuration governing the endpoint
   (security rules, transactions). Read only enough of each file to follow the calls.
5. Record the chain as an ordered, layer-labeled file list. Batch at 20 files or
   4,000 lines; if the chain exceeds that, review controller, service, and
   repository layers separately and synthesize.

---

## Step 3: Detect tech stack

Using the files in scope (not context files), classify stacks and flags. Multiple
stacks can be active.

### Stacks

- **`java`**: `*.java`/`*.kt`, `*.gradle*`, or `pom.xml`.
- **`rust`**: `*.rs` with a `Cargo.toml`, including workspace members.
- **`python`**: `*.py` with `pyproject.toml`, `setup.py`, `setup.cfg`, or `requirements*.txt` in the project.
- **`react`**: `*.tsx`/`*.ts`/`*.jsx` containing React patterns (imports from `react`, JSX, hooks).

### Flags

Use path and content signals (Grep the in-scope files; do not read them whole):

- **`security`**
  - Any stack: paths containing `Security`, `Auth`, `Filter`, `Token`, `Credential`,
    `Secret` (case-insensitive), or content with `jwt`, `oauth`, password/secret handling.
  - Java: `@PreAuthorize`, `SecurityFilterChain`, `HttpSecurity`.
  - Rust: crypto, `unsafe`, `extern`, FFI, `build.rs`, process/filesystem/outbound
    HTTP, registry or dependency-source changes.
  - Python: `*payer*`, `*wallet*`, `*payment*` paths, `subprocess`, filesystem state,
    HTTP clients, env/config handling, redaction.
- **`api-contract`**
  - Java: `@RestController`, `@*Mapping`, `@RequestBody`, `@PathVariable`,
    `@RequestParam`, `@ControllerAdvice`, `@ExceptionHandler`, `@Schema`,
    `@Operation`, `Pageable`, `@JsonProperty`, `@JsonAlias`.
  - Rust: changed `pub` items, crate exports, serde types, routes, OpenAPI, CLI
    args/output, FFI, features, or MSRV.
  - Python: FastAPI/Flask routes, Typer/Click/argparse CLIs, public `__init__.py`
    exports, request/response/schema models, error envelopes, `openapi.*`.

The primary reviewer for each stack already covers performance and concurrency,
so those need no separate flag.

---

## Step 4: Locate checklists

The checklists sit next to this SKILL.md:

- Always: [checklist-universal.md](checklist-universal.md)
- `java` → [checklist-java-spring.md](checklist-java-spring.md)
- `rust` → [checklist-rust.md](checklist-rust.md)
- `react` → [checklist-react-ts.md](checklist-react-ts.md)
- Any other `checklist-*.md` in this directory is a user-added checklist; include it.

Resolve their absolute paths and confirm they exist (`ls`); do not read them into
this session. Reviewer agents read them. **If `checklist-universal.md` is missing,
STOP and report the error.** Warn about other missing checklists and continue.

---

## Step 5: Dispatch reviewer agents

### Reviewer table

| Stack | Primary reviewer | + `security` flag | + `api-contract` flag |
|---|---|---|---|
| `java` | `greenharbor-backend-reviewer-java` | `greenharbor-backend-security-reviewer-java` | `greenharbor-backend-api-contract-reviewer-java` |
| `rust` | `greenharbor-backend-reviewer-rust` | `greenharbor-backend-security-reviewer-rust` | `greenharbor-backend-api-contract-reviewer-rust` |
| `python` | `greenharbor-backend-reviewer-python` | `greenharbor-backend-security-reviewer-python` | `greenharbor-backend-api-contract-reviewer-python` |
| `react` | `greenharbor-frontend-reviewer` | covered by primary | covered by primary |
| none detected | `general-purpose` | — | — |

### Dispatch by mode

- **DIFF / SCOPE:** for each batch, launch the primary reviewer for each detected
  stack, the flagged specialists, and one checklist agent (see below).
- **FLOW:** launch the primary reviewer with the full chain, the flagged specialists,
  and one checklist agent.
- **FULL:** launch one primary reviewer per section, with the checklist block added
  to its prompt (no standalone checklist agent). After all sections finish, launch
  one `general-purpose` synthesis agent (below).

Launch agents in parallel, at most 6 in flight; queue the rest. If the review
needs more than 3 batches or sections, print the batching plan first, then continue
batch by batch. Carry only terse findings between batches, never raw code.

### Reviewer prompt

Build every reviewer prompt from this skeleton, filling in the mode-specific
**Review input** and **Focus** blocks below:

```
## Role boundary
You are a read-only reviewer worker for a parent review workflow. Do not edit
files, spawn or delegate to agents, or invoke greenharbor-* workflow skills.
Read what you need yourself and report findings only.

## Review input
<mode-specific block>

## Focus
<mode-specific block>
For class-based code, also trace every constructor parameter, injected dependency,
and field with a setter: if it is never read or called, it is dead weight; if it
could be derived from another injected dependency, say so.

## Output
Output ONLY findings, in this exact format:

[file:line] SEVERITY Problem description
Fix: suggested fix
Files: <comma-separated list of files the fix would touch>

SEVERITY is one of CRITICAL, HIGH, MEDIUM, LOW. Only flag real problems with
evidence in the code under review. No preamble, summaries, or "looks good"
comments. If there are no findings, output exactly: NO_FINDINGS
```

**Review input — DIFF:**
```
Base commit: <BASE sha> (merge-base with <baseline>)
Changed files in this batch:
<paths>
New untracked files in this batch (read in full):
<paths, or "none">
For each changed file run `git diff <BASE> -- <file>` yourself (default -U8;
widen to -U40 only when you need more context to confirm a finding).
```

**Review input — SCOPE:**
```
Scope: <scope label>
Files to review (read them yourself):
<paths>
Context files (read only as needed to understand dependencies — do NOT review):
<paths>
```

**Review input — FULL:**
```
Section: <name> (<id>), priority <priority>, stack <stacks>
Files to review (read them yourself):
<paths>
```

**Review input — FLOW:**
```
Endpoint: <METHOD> <PATH>
Call chain, in order, with layer labels (read the files yourself):
<Controller → Service(s) → Repository/Client(s) → Entities/DTOs → Config>
Only flag issues relevant to this flow.
```

**Focus:** for primary reviewers and specialists in DIFF/SCOPE/FULL, use
"Production-readiness issues in your area of expertise; for FULL, also the
section focus: `<focus>`." For FLOW, use:

1. Authorization: is the endpoint secured? Can users reach other users' data?
2. Input validation at the boundary; injection risks.
3. Transaction boundaries; partial-commit risk.
4. Error handling and propagation at each layer.
5. Performance: N+1 queries, missing pagination, unbounded result sets.
6. Data consistency: races, check-then-act without atomicity.
7. Response leakage: internal IDs, stack traces, sensitive fields.

### Checklist agent (DIFF, SCOPE, FLOW)

Launch in the same message as the reviewers: `subagent_type: general-purpose`,
`model: sonnet`. Use the reviewer prompt skeleton with the same Review input and
this Focus:

```
Read and apply these checklists: <absolute checklist paths>
Pass 1 (CRITICAL): apply every CRITICAL check.
Pass 2 (INFORMATIONAL): apply every INFORMATIONAL check.
<if --critical-only: "Skip Pass 2 entirely.">
Honor each checklist's Suppressions section — never flag suppressed items.
```

### FULL-mode checklist block

Append to each section reviewer's Focus:

```
Also read and apply these checklists to the files in this section:
<absolute checklist paths>
<if --critical-only: "Apply only the CRITICAL checks.">
Honor each checklist's Suppressions section. Use the same output format.
```

### FULL-mode synthesis agent

```
## Role boundary
<same as the reviewer skeleton>

## Task
You have findings from focused review agents across these sections:
<section names and file lists>

Findings:
<all section findings, concatenated>

Perform a cross-cutting synthesis, reading code only to confirm a pattern:
1. INCONSISTENCIES: patterns used in one section but not a sibling (e.g., input
   validation in one service but not another).
2. DEPENDENCY RISK: the most-depended-on components that lack defensive coding.
3. MISSING INTEGRATION: section boundaries with no integration test coverage.
4. ARCHITECTURAL CONCERNS: systemic issues visible only across sections.

Output only new findings, in the same format as the section reviewers.
```

**Agent failures are not fatal.** If an agent fails or times out, retry it once;
if it fails again, continue with the other results and note the gap in the report.

---

## Step 6: Merge, verify, classify, and output

### 6a. Deduplicate and map severity

1. **Deduplicate:** when findings refer to the same issue at the same location,
   merge them and keep the most specific wording and fix.
2. **Map severity:** agent `CRITICAL`/`HIGH` → `CRITICAL`; `MEDIUM`/`LOW` → `INFORMATIONAL`.
3. If `--critical-only`, drop INFORMATIONAL findings.

### 6b. Verify CRITICAL findings

Agents report false positives as confidently as real issues. Before reporting,
verify each CRITICAL finding yourself: read the cited lines with about 10 lines of
context (in DIFF mode, `git diff <BASE> -U10 -- <file>` when the issue is in
changed lines) and confirm the problem exists in the current code.

- Confirmed → keep.
- Already handled, wrong, or not reachable → drop it and count it under
  "dropped on verification" in the summary.
- Plausible but unprovable from the code → keep, marked `(unverified)`.

Verify at most 15 findings; mark any beyond that `(unverified)`. Do not verify
INFORMATIONAL findings.

### 6c. Classify each finding by resolution path

A finding is **DESIGN** (`/greenharbor-plan-work`) if ANY of these apply:
- It requires a new abstraction, pattern, or architectural layer.
- It affects multiple files across different packages/modules (cross-cutting).
- The fix is a choice between competing approaches with trade-offs.
- It is a missing capability that needs requirements thinking (e.g., "no rate
  limiting", "no retry strategy", "no auth on this endpoint group").
- It requires a schema migration or API contract change.
- The problem is clear but the solution needs exploration.
- It redesigns a flow or restructures responsibilities between classes.

A finding is **IMPL** (`/greenharbor-orchestrate`) if ALL of these apply:
- The fix is well-defined and surgical — you know exactly what to change.
- It is contained to 1-3 files with no architectural decisions.
- Examples: add a missing `@Valid`, fix an N+1 with `JOIN FETCH`, add a null
  check, add a missing `AbortController` cleanup, switch to a parameterized
  query, add `readOnly = true` to a `@Transactional`.

When in doubt, classify as DESIGN.

### 6d. Group findings into parallel waves

Waves are sets of findings whose fixes touch disjoint files, so they can be
worked in parallel.

1. **File footprint:** IMPL → the finding's `Files:`. DESIGN → `Files:` plus files
   named in the "Why design needed" rationale. If the footprint is unclear or
   cross-cutting, mark it `unbounded`.
2. Within each resolution path (DESIGN and IMPL separately), take findings in
   severity order (CRITICAL first). Put each `unbounded` finding in its own wave.
   Put every other finding in the first existing wave whose claimed files do not
   overlap its footprint; if none fits, open a new wave.
3. Number waves across the report: DESIGN waves first, then IMPL waves.
4. **Consolidate:** within a wave, merge findings that share a theme or module into
   one workload — one `/greenharbor-orchestrate` command per IMPL workload, one
   `/greenharbor-plan-work` command per DESIGN workload. Findings in different
   subsystems stay separate.

Waves are a planning hint, not a hard constraint.

### 6e. Build the report

**Header by mode:**
- DIFF: `Diff Review (<baseline>): N issues (X critical, Y informational)`
- SCOPE: `Scope Review [<label-1>, <label-2>, ...]: N issues (X critical, Y informational)`
- FULL: `Full Project Review: N issues (X critical, Y informational) across M sections`
- FLOW: `Flow Review (<method> <path>): N issues (X critical, Y informational)`

**Format.** Group by resolution path, then by scope label or section when relevant,
then by severity:

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
Dropped on verification: N | Unverified: N | Agent failures: <none or list>

## Action Playbook
<per wave: `/greenharbor-plan-work "<description>" --review {REPORT_PATH}` for
DESIGN workloads, `/greenharbor-orchestrate {IMPL_PLAN_PATH} --scope "Wave N"` for
IMPL workloads>
```

Mode additions: SCOPE prefixes findings with the scope label; FULL prefixes
findings with the section and lists synthesis findings separately; FLOW adds
`Flow chain: <the traced chain>` above the findings.

Branch propagation: if `--branch`/`--baseline` was given, append `--branch <ref>`
to playbook commands, stripping any `origin/` prefix.

If no issues were found: `<Report Header>: No issues found.`

### 6f. Write the report

SCOPE, FULL, and FLOW always write a report: `--out` if given, otherwise
`reports/greenharbor-code-review-<mode>-<YYYY-MM-DD>.md` (single-target SCOPE:
`reports/greenharbor-code-review-scope-<scope-label>-<YYYY-MM-DD>.md`). DIFF mode
writes one only when `--out` is given or an IMPL mini-plan will be generated
(implicit path `reports/greenharbor-code-review-diff-<YYYY-MM-DD>.md`). If the
target file already exists, append `-2`, `-3`, … instead of overwriting. Create
`reports/` if needed. Print the report to the conversation in every mode.

### 6g. Write the IMPL mini-plan

Generate a mini-plan when CRITICAL IMPL findings exist, or for INFORMATIONAL-only
IMPL findings when `--out` was given. Otherwise skip this step.

It restructures the IMPL findings into the `## Wave N:` / `### W{N}-{NN}:` format
that `/greenharbor-orchestrate` parses, so the report stays human-oriented and the
mini-plan machine-oriented.

**Path:** `reports/greenharbor-code-review-impl-plan-<YYYY-MM-DD>.md` (SCOPE:
`reports/greenharbor-code-review-impl-plan-<scope-label>-<YYYY-MM-DD>.md`), with
the same no-overwrite suffix rule.

```markdown
# IMPL Fixes from Review

**Source:** `{REPORT_PATH}`
**Generated:** <date>
**Branch:** `<ref>` (pass `--branch <ref>` to `/greenharbor-orchestrate`)  ← only if a non-default baseline was used

## Summary
Surgical fixes identified by /greenharbor-code-review. Each work unit is a
self-contained fix that needs no architectural decisions.

## Risk Flags
`security`: <yes if any unit fixes a security finding, else no> | `public-api`: no | `migration`: no

## Wave 1: <wave theme from the Action Playbook>

### W1-01: <I{N} finding title, imperative form>
<problem description from the finding>

**Files:** <files from the finding's Files: field>
**Acceptance criteria:**
- <the finding's Fix: restated as a checkable condition>
- Existing tests continue to pass
**Error handling:** N/A (surgical fix)
**Tests:** Add a regression test that fails before the fix; run the affected tests

## Wave 2: <next wave theme>
...
```

Mapping: each IMPL wave in the playbook becomes a `## Wave N:` (numbered from 1,
independent of the report's mixed numbering); each IMPL finding becomes one
`### W{N}-{NN}:` unit, including findings that were consolidated into one command.

After writing, print: `IMPL plan written to: <path> (<N> work units across <M> waves)`

---

## Step 7: Handle critical issues (DIFF mode only)

In SCOPE, FULL, and FLOW modes the report is the deliverable — skip this step.

If `--fix` is set:
- Apply the fix for each verified CRITICAL **IMPL** finding. Never auto-fix
  DESIGN findings or `(unverified)` findings.
- Then run the narrowest test command covering the touched files, if the project
  makes one obvious; do not run the full suite. Report the result.
- Output a summary of what changed. Do NOT commit, push, or create PRs.

If `--interactive` is set and `--fix` is not:
- For EACH critical **IMPL** finding, ask a separate `AskUserQuestion` with the
  problem and fix. Options: **Fix it now**, **Acknowledge**, **False positive — skip**.
- For EACH critical **DESIGN** finding, ask with the problem and why it needs
  design. Options: **I'll run /greenharbor-plan-work next**, **Acknowledge**,
  **False positive — skip**.
- Apply the chosen IMPL fixes, then run targeted tests as for `--fix`.
- For DESIGN findings marked for planning, print the ready-to-run
  `/greenharbor-plan-work` command (with `--review {REPORT_PATH}` if a report was
  written and `--branch <ref>` if a non-default baseline was used). Do not invoke
  it yourself.

Default (neither flag): report only. Do not prompt or modify files.

---

## Important Rules

- **Read-only by default.** Modify files only under `--fix` or a "Fix it now"
  answer. Never commit, push, or create PRs.
- **Stay token-aware.** Metadata in this session; code and diffs stay with the agents.
- **Verify before reporting.** Every CRITICAL finding is confirmed against the
  code or marked `(unverified)`.
- **Be terse.** One line problem, one line fix. No preamble.
- **Only flag real problems.** Skip anything that is fine or already addressed.
- **Respect scope.** In SCOPE mode, flag only resolved files, not context files.
  In FLOW mode, flag only issues relevant to the traced flow.
