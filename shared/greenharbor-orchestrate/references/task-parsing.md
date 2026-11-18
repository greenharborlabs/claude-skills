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

The checkpoint file is the task graph; there is no separate task tool. Write it
on every non-dry run (not only `--resume`) so a later `--resume` has state.
Record each work unit's status and its dependencies in `depends_on`; re-read
unit bodies from the plan instead of storing them.

Checkpoints:

- Store checkpoints in `.orchestrate/<plan-file-basename>-<plan-hash>.json`.
- If the checkpoint path would collide, prefer the matching `plan_path` and
  `plan_hash`; otherwise create a new checkpoint.
- Keep checkpoint state compact:

```json
{
  "plan_path": "<path>",
  "plan_hash": "<content hash>",
  "start_commit": "<START_COMMIT>",
  "tasks": {"<id>": "pending|in_progress|implemented|completed|blocked"},
  "depends_on": {"<id>": ["<id>"]},
  "blocked_reasons": {"<id>": "<short reason>"},
  "gates": {"final_verification": "pending|completed", "specialist_review": "pending|completed"},
  "touched_tests": ["<path>"],
  "contract_registry": [{"file": "<path>", "name": "<symbol>", "signature": "<compact signature>"}],
  "agent_suite": {"coder": "<agent>", "reviewer": "<agent>", "architect": "<agent>"},
  "test_cmd": "<literal command template>"
}
```

On `--resume`:

- `completed`: skip.
- `implemented`: skip re-implementation; it still goes through the final
  cumulative review.
- `in_progress`: reset to pending.
- `pending`: reuse.
- `blocked`: keep blocked unless its dependency context changed or the user
  scopes directly to that task.
- Add only work units not already in the checkpoint.
- If all work units are implemented or completed, resume at the first
  incomplete gate in `gates`.

Update the checkpoint after each work group is implemented or blocked, after
each wave's contract registry update, and on every gate change. Do not store raw
diffs, full reports, or file contents.

## Dry Run

Print and stop:

```text
| Batch | Task ID | Subject | Dependencies | Estimated Files |
| --- | --- | --- | --- | --- |

Agent suite   : <coder> / <reviewer> / <architect>
Test command  : <TEST_CMD>
Specialists   : <security + api-contract, or none>
Est. min spawns: <lower bound before fix cycles>
References    : task-parsing, stack-detection, agent-prompts, wave-execution, review-loop, specialist-reviews only if needed
```
