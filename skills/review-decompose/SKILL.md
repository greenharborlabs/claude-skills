---
name: review-decompose
description: "Decompose a project into focused review sections and write each as a separate .md file. This skill should be used when the user wants to create a structured code review plan, decompose a codebase for parallel review, or says 'review-decompose'. It analyzes project structure, splits it into cohesive sections by concern, and produces per-section review files with objectives, file lists, and sub-agent prompt templates."
---

# Review Decompose

## Overview

Analyze a project's structure and decompose it into discrete, reviewable
sections. Each section is written to its own .md file with review objectives,
file lists, and a ready-to-use sub-agent review prompt. Enables focused,
parallel code reviews by specialized reviewer agents.

## Help

If the argument is `--help`, `help`, or no arguments are provided, print this
usage summary and stop (do not run the workflow):

```
/review-decompose - Decompose a project into focused review sections

Usage:
  /review-decompose                                  Review current project, output to plans/review/
  /review-decompose --out path/to/output/            Override output directory
  /review-decompose --agent nanoclaw-reviewer         Override agent type

Arguments:
  --out         Output directory for review .md files (default: plans/review/)
  --agent       Agent type for decomposition and sub-agent prompts (default: nanoclaw-architect)

Examples:
  /review-decompose
  /review-decompose --out docs/reviews/
  /review-decompose --agent nanoclaw-reviewer

Output: One .md file per review section + 00-master-plan.md with the full decomposition
```

## Arguments

Parse the invocation arguments as follows:

- **`--out` (optional):** Output directory for review files. Default: `plans/review/`
- **`--agent` (optional):** Agent type for decomposition and sub-agent prompt templates. Default: `nanoclaw-architect`

## Workflow

### Step 1: Gather Project Structure

Read the project's directory tree, key configuration files (package.json,
tsconfig.json, pom.xml, application.yml, docker-compose.yml, etc.), and
CLAUDE.md or README if present. Build a mental model of the project's
architecture, layers, and boundaries.

### Step 2: Decompose into Review Sections

Split the project into discrete, cohesive review sections following these rules:

1. Each section maps to a single logical concern — do not mix orchestration
   logic with data persistence
2. No section should require more than ~15-20 files; split further if needed
3. Identify cross-cutting concerns (logging, error handling, security) as their
   own section
4. If agent tools/plugins exist, each tool family gets its own section
5. Order sections so foundational layers (core models, interfaces) are reviewed
   before dependent layers

Assign each section one of these focus areas:
- Core Agent Logic
- Tool/Plugin Layer
- Orchestration and Routing
- Data/State Management
- API and Integration Layer
- Configuration and Infrastructure
- Testing and Quality
- Cross-Cutting Concerns

For each section, determine:
- **id:** Sequential identifier (e.g., `section-001`)
- **name:** Short descriptive name
- **focus_area:** One of the focus areas above
- **files:** List of relevant file paths
- **review_objectives:** Specific things the reviewer should evaluate
- **dependencies:** IDs of sections that should be reviewed first
- **suggested_agent_persona:** Brief description of the ideal reviewer
- **priority:** high, medium, or low
- **estimated_complexity:** low, medium, or high

Also produce shared context that all reviewers need:
- **architecture_summary:** Brief summary all sub-agents should know
- **critical_interfaces:** Key interfaces/contracts that cross section boundaries
- **known_risks:** Any red flags visible from the top-level structure

### Step 3: Write Master Plan

Write `00-master-plan.md` to the output directory containing:

1. The full decomposition as a JSON block following this schema:

```json
{
  "sections": [
    {
      "id": "section-001",
      "name": "Short descriptive name",
      "focus_area": "One of the focus areas",
      "files": ["list/of/relevant/file/paths"],
      "review_objectives": [
        "Specific thing sub-agent should evaluate"
      ],
      "dependencies": ["ids of sections this depends on"],
      "suggested_agent_persona": "Brief reviewer persona description",
      "priority": "high | medium | low",
      "estimated_complexity": "low | medium | high"
    }
  ],
  "review_order": ["section-001", "section-002"],
  "shared_context": {
    "architecture_summary": "Brief summary all sub-agents should know",
    "critical_interfaces": ["Key cross-boundary interfaces"],
    "known_risks": ["Red flags from top-level structure"]
  }
}
```

2. The recommended review order with rationale

### Step 4: Write Per-Section Review Files

For each section, write a numbered .md file (e.g., `01-shared-types-and-contracts.md`,
`02-data-state-management.md`) to the output directory. Each file contains:

1. Section metadata (name, focus area, priority, complexity)
2. Review objectives as a numbered list
3. Files in scope as a list
4. Dependencies on other sections
5. Shared context (architecture summary, critical interfaces)
6. A ready-to-use sub-agent review prompt in this format:

```
## Sub-Agent Review Prompt

You are a [suggested_agent_persona]. You are performing a focused code review
of the "[name]" section of a [project type] project.

Your review objectives are:
[review_objectives as numbered list]

Files in scope:
[files list]

Shared context you must be aware of:
[shared_context.architecture_summary]
[shared_context.critical_interfaces]

Perform your review and return findings as:
- ISSUES: (bugs, anti-patterns, violations)
- SUGGESTIONS: (improvements, optimizations)
- QUESTIONS: (ambiguities needing clarification from the team)
- SCORE: (1-10 confidence this section is production-ready, with rationale)
```

### Step 5: Summary

After writing all files, print a summary listing:
- Total sections created
- Review order (by priority)
- Output directory path
- Suggested next step: how to run the reviews (e.g., spawn reviewer agents
  per section, or use `/orchestrate` with an action plan derived from the reviews)
