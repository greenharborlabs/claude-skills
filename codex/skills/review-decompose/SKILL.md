---
name: review-decompose
description: Decompose a project into focused review sections and write each as a separate markdown file. Use when the user wants a structured code review plan, wants to decompose a codebase for focused review, or says "review-decompose".
---

# Review Decompose

Analyze a project's structure and decompose it into discrete, reviewable sections. Write a master plan and per-section review files that can be used by humans, Codex, or explicitly requested parallel agents.

## Help

If the argument is `--help`, `help`, or no arguments are provided, show:

```text
review-decompose - Decompose a project into focused review sections

Usage:
  review-decompose
  review-decompose --out plans/review/
```

## Arguments

- `--out PATH`: output directory for review files. Default `plans/review/`.

## Workflow

1. Gather project structure:
   - Directory tree.
   - Key configs and manifests.
   - README, AGENTS.md, CLAUDE.md, docs, or specs if present.
2. Decompose into cohesive sections:
   - One logical concern per section.
   - Target 15-20 files max per section.
   - Cross-cutting concerns can be separate sections.
   - Foundational layers should precede dependent layers.
3. Assign each section:
   - `id`
   - `name`
   - `focus_area`
   - `files`
   - `review_objectives`
   - `dependencies`
   - `suggested_reviewer_persona`
   - `priority`
   - `estimated_complexity`
4. Write `00-master-plan.md` to the output directory with a JSON decomposition block and review order.
5. Write one numbered per-section markdown file containing metadata, objectives, files, dependencies, shared context, and a ready-to-use review prompt.
6. Summarize the section count, review order, output directory, and next step.

## Focus Areas

- Core Domain Logic
- Tool/Plugin Layer
- Orchestration and Routing
- Data/State Management
- API and Integration Layer
- Configuration and Infrastructure
- Testing and Quality
- Cross-Cutting Concerns

## Codex Agent Handling

Do not assume named external reviewer agents exist. The per-section prompts should be self-contained and usable by a human reviewer or by Codex if the user explicitly requests parallel agent review.

## Master Plan Schema

```json
{
  "sections": [
    {
      "id": "section-001",
      "name": "Short descriptive name",
      "focus_area": "Focus area",
      "files": ["list/of/relevant/file/paths"],
      "review_objectives": ["Specific review objective"],
      "dependencies": ["section ids"],
      "suggested_reviewer_persona": "Brief reviewer persona",
      "priority": "high | medium | low",
      "estimated_complexity": "low | medium | high"
    }
  ],
  "review_order": ["section-001"],
  "shared_context": {
    "architecture_summary": "Brief summary",
    "critical_interfaces": ["Key cross-boundary interfaces"],
    "known_risks": ["Red flags"]
  }
}
```
