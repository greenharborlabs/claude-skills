---
name: greenharbor-improve-codebase-architecture
description: Explore a codebase to find opportunities for architectural improvement, focusing on testability and deepening shallow modules. Use when the user wants to improve architecture, find refactoring opportunities, consolidate tightly coupled modules, or make a codebase more AI-navigable.
---

# Improve Codebase Architecture

Explore a codebase, surface architectural friction, and propose module-deepening refactors as RFC-style GitHub issues or local issue drafts.

A deep module has a small interface hiding a large implementation. Deep modules are easier to test at their boundaries and easier for agents to navigate.

## Workflow

1. Explore the codebase read-only. Notice friction:
   - Understanding one concept requires bouncing across many files.
   - Modules are so shallow that callers must understand implementation detail.
   - Pure functions were extracted for testability but important bugs live in orchestration.
   - Coupled modules create integration risk between layers.
   - Important code is untested or hard to test through public interfaces.
2. Read `references/deep-module-refactor-rfc.md` for dependency categories and the issue template.
3. Present 3-7 candidates. For each:
   - Cluster: modules/concepts involved.
   - Coupling: shared types, call patterns, co-owned state, or lifecycle coupling.
   - Dependency category from the reference.
   - Test impact: current fragile tests and proposed boundary tests.
4. Ask the user which candidate to explore.
5. For the chosen candidate, design at least three materially different interfaces yourself:
   - Minimal interface.
   - Flexible/extensible interface.
   - Common-case optimized interface.
   - Ports/adapters interface when crossing infrastructure boundaries.
6. Compare the designs, recommend one, and state why.
7. After the user picks or accepts a recommendation, create a GitHub issue if the GitHub CLI/app is available and the user has not prohibited it. Otherwise write a local RFC draft and report the path.

## Codex Agent Handling

Do not assume named external agents exist. If the user explicitly asks for parallel agents, Codex worker or explorer agents may be spawned with self-contained prompts for independent design alternatives. Otherwise, perform the design alternatives directly in separate sections.

## Output Standard

- Ground every candidate in concrete file paths.
- Prefer fewer, deeper refactors over broad rewrites.
- Keep implementation proposals as RFCs, not code changes, unless the user separately asks to implement.
- Focus on public interfaces and test seams that reduce caller complexity.

## Resources

- `references/deep-module-refactor-rfc.md`: dependency categories and RFC issue template.
