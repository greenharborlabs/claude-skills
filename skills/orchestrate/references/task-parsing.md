# Task Parsing

Load this after pre-flight and before building the task graph.

## Plan Extraction

Read the plan file. Extract work units with the first matching rule:

1. Wave/phase headings with H3 sub-items:
   - `## Wave N:` or `## Phase N:` containing `### ID:` sub-items.
   - Each H3 is one work unit.
   - Units in Wave N depend on all units in Wave N-1.
   - Units within the same wave are independent.
2. Numbered question/decision blocks:
   - `### Q{N}:` or `### {N}.`.
   - Sequential unless dependency language says otherwise.
3. JSON task array:
   - Fenced JSON array of `{id, name, depends_on?}` objects.
4. Markdown checklist:
   - `- [ ]` pending, `- [x]` completed.
   - Sequential by default.
5. H2 fallback:
   - Each `## Heading` is one unit, sequential.

Apply `--scope` after extraction and always include transitive dependencies.

## Scope Syntax

- Heading match: case-insensitive substring against H2/H3 headings.
- Range: `"Heading A-Heading B"` inclusive.
- ID list: `"C-01,C-02,Q5"` comma-separated.

## Task Graph

For each work unit, create or reuse a task:

- `subject`: imperative heading.
- `description`: full unit body.
- `activeForm`: present-continuous form.

Wire dependencies with blocked-by/blocks relationships.

Resume mode:

- `completed`: skip.
- `in_progress`: reset to pending.
- `pending`: reuse.
- Create only unmatched work units.

Create sentinel tasks:

- `final-verification-gate`.
- `specialist-review-gate`.

If work-unit tasks are complete on resume, resume at the first incomplete gate.

## Dry Run

Print and stop:

```text
| Batch | Task ID | Subject | Dependencies | Estimated Files |
| --- | --- | --- | --- | --- |

Agent suite   : <coder> / <reviewer> / <architect>
Test command  : <TEST_CMD>
Specialists   : <security + api-contract, or none>
Est. min spawns: <lower bound before fix cycles>
References    : task-parsing, stack-detection, agent-prompts, review-loop, specialist-reviews only if needed
```
