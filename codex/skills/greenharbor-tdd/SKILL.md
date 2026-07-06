---
name: greenharbor-tdd
description: Test-driven development with a red-green-refactor loop. Use when the user wants to build features or fix bugs using TDD, mentions "red-green-refactor", wants integration tests, or asks for test-first development.
---

# Test-Driven Development

## Philosophy

Tests should verify behavior through public interfaces, not implementation details. Code can change entirely; tests should continue to describe what the system does.

Good tests are integration-style when practical: they exercise real code paths through public APIs and read like specifications. Bad tests are coupled to private methods, internal collaborators, or incidental object shape.

Read these references as needed:

- `references/tests.md`: examples and test quality guidance.
- `references/mocking.md`: mocking rules.
- `references/interface-design.md`: designing testable public interfaces.
- `references/deep-modules.md`: deep module guidance.
- `references/refactoring.md`: refactor candidates after green.

## Anti-Pattern: Horizontal Slices

Do not write all tests first and then all implementation. Use vertical slices:

```text
RED -> GREEN: test 1 -> implementation 1
RED -> GREEN: test 2 -> implementation 2
RED -> GREEN: test 3 -> implementation 3
```

Each test should respond to what was learned in the previous cycle.

## Workflow

1. Plan the public behavior:
   - Identify the public interface.
   - List behaviors to test, ordered by risk and user value.
   - Prefer a deep module boundary when the code is complex.
2. Tracer bullet:
   - Write one test for one behavior.
   - Run it and confirm it fails for the expected reason.
   - Implement the smallest code change that makes it pass.
3. Incremental loop:
   - Add one behavior test at a time.
   - Run the focused test.
   - Implement only enough production code to pass.
   - Keep tests on observable behavior.
4. Refactor:
   - Refactor only when green.
   - Remove duplication, deepen modules, simplify interfaces, and improve names.
   - Run focused tests after each refactor step.
5. Final verification:
   - Run the relevant test class/module.
   - Run broader checks only when the blast radius warrants it.

## Per-Cycle Checklist

```text
[ ] Test describes behavior, not implementation
[ ] Test uses public interface only
[ ] Test would survive internal refactor
[ ] Failure was observed before implementation
[ ] Code is minimal for this test
[ ] No speculative features added
```

## Collaboration

If requirements are ambiguous, ask the user only about public behavior or priorities that change the test plan. Do not ask about details discoverable from the codebase.
