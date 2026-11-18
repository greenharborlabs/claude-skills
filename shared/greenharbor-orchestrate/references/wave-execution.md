# Wave Execution

Load this at the start of the first wave, together with agent-prompts.md. It
governs everything between dispatching Coders and the final verification gate.

## Batch Exploration

Before each batch:

- Re-read the relevant wave/section from the plan file.
- Max `2 file reads per task + 2 baseline`.
- Max `1 search per task + 1 baseline`.
- Prefer targeted line ranges.
- Use the contract registry instead of re-reading prior batch interfaces.
- Extract acceptance criteria into IDs derived from the task ID
  (`W1-01/AC-1`, `W1-01/AC-2`, ...) so the same IDs can be re-extracted from the
  plan at final review. Preserve the exact criterion text in that list;
  reference IDs elsewhere instead of repeating long prose.

For each task identify only entry points, required signatures/contracts, and
affected test files.

Group sizing:

- Default one task = one work group.
- Merge tasks sharing the same entry-point file.
- Max 3 tasks per work group.
- Max 3 running sub-agents total.

## Bounded Sub-Agent Queue

Never have more than 3 sub-agents running across Coders, Reviewers, Architects,
specialists, and docs workers.

- Claude Code: send at most 3 Agent calls in one message, wait for all of them
  to return, then send the next batch.
- Codex: track an explicit `ACTIVE_AGENTS` set. Before spawning with 3 active,
  wait until at least one completes, capture its report, close it, and remove
  it from the set. Add each new agent ID immediately. Refill one slot at a time.

Do not launch the whole wave at once when it has more than 3 work groups.

## Wave Checklist

Before a batch, print:

```text
Wave N checklist:
- [ ] Task-ID: subject
```

After each Coder reports completion, mark the task as implemented and print the
updated checklist. Do not mark tasks complete until the cumulative review and
final checks pass.

## Coder Failure Handling

If a Coder times out, crashes, or omits the expected completion report:

1. Log raw output/error.
2. Retry once with the same prompt.
3. If still invalid, mark blocked and continue only where dependencies allow.

If `SELF_TEST: FAIL` (the Coder already used its own fix attempts), mark the
work group blocked with the failure summary. Finish the rest of the current
wave, then stop at the wave completion summary even under `--no-pause`, because
the next wave depends on every unit in this one. Offer: one focused fix Coder,
continue anyway, or stop.

## Contract Registry Update

When a wave finishes, before the wave summary:

- Add up to 5 `NEW_INTERFACES` entries per work group from Coder reports:
  `{ file, name, signature }`.
- If a changed file already has registry entries and the Coder did not report
  updated interfaces, remove stale entries.
- Write the updated registry and task status to the checkpoint.

Later-wave Coder prompts get relevant entries under `Cross-batch interfaces`.

## Wave Completion Summary

After all tasks in a wave are implemented, print a concise
implementation-progress summary in this shape:

```text
Wave N complete.

Business / manager summary:
- What completed:
- What it solved:
- Why it mattered:

Developer / technical summary:
- What changed:
- Tests / review: Coder self-tests <pass/fail counts>; review deferred to final cumulative gate
- Technical impact:
```

Keep each bullet short unless detail is needed for a decision, blocker, or
material risk. Do not paste raw diffs or long agent reports. Wait for the user
to confirm before advancing to the next wave, unless `--no-pause` is set and
nothing in the wave is blocked.

## Context Hygiene

After each wave, discard raw file contents, Coder reports, and spec excerpts.
Carry forward only task status, blocked reasons, `START_COMMIT`, agent suite,
`TEST_CMD`, touched tests, contract registry, and the concise wave completion
summary.
