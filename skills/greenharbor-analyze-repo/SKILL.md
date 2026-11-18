---
name: greenharbor-analyze-repo
description: Perform a comprehensive, multi-phase technical analysis of a git repository. This skill should be used when the user wants to understand a codebase deeply, analyze repository architecture, generate a technical report, or says 'greenharbor-analyze-repo'. It launches parallel sub-agents to examine project structure, components, data flow, configuration, AI patterns, code quality, and provides onboarding guidance. Triggers on "analyze repo", "analyze this repo", "analyze codebase", "repo analysis", "codebase analysis", "technical report".
allowed-tools:
  - Read
  - Grep
  - Glob
  - Bash
  - Write
  - Agent
---

# Analyze Repo

## Overview

Generate a comprehensive technical analysis report for the current git repository. The analysis covers 8 phases: high-level overview, core components deep dive, data flow analysis, configuration & integrations, AI/LLM patterns, code quality assessment, setup & extension guide, and summary insights.

Protect the context window: sub-agents read the code and return condensed findings; this session verifies the high-stakes claims and writes the report.

## Help

If the argument is `--help` or `-h`, display the following help text and stop (do not run the analysis):

```
greenharbor-analyze-repo — Comprehensive multi-phase git repository analysis

USAGE
  /greenharbor-analyze-repo [OPTIONS] [OUTPUT_FILE]

ARGUMENTS
  OUTPUT_FILE        Path for the generated report (default: analysis-report.md)

OPTIONS
  --help, -h         Show this help message
  --phases LIST      Comma-separated phase numbers to run (default: all)
                     Example: --phases 1,2,6
  --quick            Run a faster analysis (skip Phase 3 data-flow tracing)

PHASES
  1  High-Level Overview      Purpose, tech stack, architecture, entry points, directory structure
  2  Core Components          Per-module deep dive with file:line references and design patterns
  3  Data Flow & Execution    Startup sequence, operation traces, ASCII flow diagrams
  4  Configuration            Env vars, config files, integrations, plugin mechanisms
  5  AI / LLM Analysis        Model providers, prompts, agent orchestration, memory management
  6  Code Quality             Strengths, tech debt, security concerns, test coverage, docs
  7  How to Run & Extend      Setup instructions, extension guide, onboarding file list
  8  Summary & Key Insights   Architectural decisions, novel aspects, recommended next steps

EXAMPLES
  /greenharbor-analyze-repo                              Full analysis → analysis-report.md
  /greenharbor-analyze-repo report.md                    Full analysis → report.md
  /greenharbor-analyze-repo --quick                      Skip data-flow tracing for speed
  /greenharbor-analyze-repo --phases 1,2,6               Only run overview, components, and quality phases
  /greenharbor-analyze-repo --phases 1,6 quick-check.md  Partial analysis to a custom file
```

## Arguments

| Argument | Description | Default |
|----------|-------------|---------|
| `OUTPUT_FILE` | Path for the generated report | `analysis-report.md` |
| `--help`, `-h` | Show help text and exit | — |
| `--phases LIST` | Comma-separated phase numbers (1-8) to include | all phases |
| `--quick` | Skip Phase 3 (data-flow tracing) for faster results | off |

Parse flags first (`--help`, `--quick`, `--phases`); any remaining positional argument is the output file path.

## Workflow

### Step 0: Preparation

1. If `--help` or `-h` is present, display the help text above and stop.
2. Parse flags and the output path. If the output file already exists, append `-2`, `-3`, … rather than overwriting.
3. Confirm this is a git repository with `git rev-parse --show-toplevel` (works in worktrees and subdirectories) and analyze from that root.
4. Capture the repository name from `git remote get-url origin` or the root directory name.
5. Determine which agents to launch from the selected phases (mapping below). Launch only agents whose phases are selected.

**Phase → Agent mapping:**
| Phases needed | Agent to launch | Agent type |
|---------------|-----------------|------------|
| 1 | Agent 1 (Structure & Stack) | `Explore` |
| 2, or 3 without `--quick` | Agent 2 (Core Components) | `Explore` |
| 3 (skipped if `--quick`) | Agent 5 (Data Flow) | `Explore` |
| 4, 5 | Agent 3 (Config, Integration & AI) | `Explore` |
| 6, 7 | Agent 4 (Quality, Tests & Docs) | `general-purpose` |
| 8 | No agent — synthesized from the other phases | — |

`Explore` agents locate and summarize code; quality, security, and test assessment needs an agent that reads and judges code, so Agent 4 is `general-purpose`.

### Step 1: Parallel Discovery

Launch the selected agents among 1-4 in a single message so they run concurrently. Ask `Explore` agents for "very thorough" search. End every agent prompt with:

> Do not edit files or spawn agents. Return condensed findings (aim for under 800 words): tables and bullets, every claim with a `path:line` reference. Cover every major module; in large repositories, sample representative files per module and say which you sampled.

**Agent 1 — Structure & Stack Discovery:**
> Explore the repository to determine: (a) project purpose from README, CLAUDE.md, package.json, or equivalent; (b) complete tech stack with versions from package.json, requirements.txt, pyproject.toml, Cargo.toml, go.mod, pom.xml, build.gradle, or equivalent dependency files; (c) architecture style based on directory structure and module organization; (d) all entry points (main files, scripts, CLI commands, server start); (e) annotated directory tree of top-level structure.

**Agent 2 — Core Components Inventory:**
> Explore every major module/component in the repository. For each, report: (a) name and file path(s); (b) responsibility; (c) key classes, functions, interfaces with file:line references; (d) internal and external dependencies; (e) design patterns used (Factory, Strategy, Observer, etc.).

**Agent 3 — Configuration, Integration & AI Analysis:**
> Explore the repository for: (a) all configuration mechanisms (env vars, config files, CLI args) with file locations; (b) external API/service/database integrations and how they authenticate; (c) plugin or extension mechanisms; (d) any LLM/AI agent usage — identify model providers, prompt structures, agent orchestration patterns, tool-use patterns, and context/memory management. Report secret names only, never secret values.

**Agent 4 — Quality, Tests & Documentation Assessment:**
> Assess the repository for: (a) code quality strengths — well-designed patterns, elegant implementations; (b) weaknesses and technical debt — fragile, incomplete, or overly complex areas; (c) security concerns — hardcoded secrets, unsafe inputs, insecure patterns (report locations, never secret values); (d) test coverage — test files, what is covered, what is missing; (e) documentation quality — README completeness, inline comments, API docs. Also determine: (f) how to set up and run the project locally; (g) how a developer would add a new feature; (h) the most important files for onboarding. Read the code you judge; do not infer quality from file names.

### Step 2: Data Flow Analysis

**Skip this step if `--quick` is set or Phase 3 is not selected.**

After Step 1 completes, launch Agent 5 (`Explore`, "very thorough") with Agent 2's component list pasted in as a starting map:

**Agent 5 — Execution & Data Flow Tracing:**
> Using this component map as a starting point: <Agent 2's component table>. Trace the execution path of this application from startup to a key operation. (a) Map the startup sequence — what happens when the application starts, in order; (b) trace a key user-facing operation end-to-end through the codebase; (c) identify how data is transformed as it moves between components; (d) note all async, concurrent, or event-driven patterns (event emitters, message queues, callbacks, promises, streams). Create ASCII diagrams showing the flow.

### Step 3: Verify Key Claims

Agents report errors as confidently as facts. Before writing, spot-check by reading the cited lines yourself (max ~10 targeted reads):

- every security concern from Agent 4,
- the entry points and run commands that go into Phase 7,
- any claim that would surprise a maintainer (e.g., "no tests for X", "unused module").

Drop or correct claims that do not hold; keep unverifiable ones only with "(unverified)".

### Step 4: Synthesize and Write Report

1. Read [references/report-template.md](references/report-template.md) for the report structure.
2. Synthesize agent findings into the template, including only the selected phases (Phase 8 draws on whichever phases ran).
3. Replace all placeholders with actual data; every observation cites file names, symbols, and line numbers.
4. For a selected phase that does not apply (e.g., Phase 5 with no AI/LLM usage), keep a one-line "Not applicable" note with the evidence.
5. Write the report to the output path and print its path plus the three most important findings.

## Quality Standards

- **Specificity**: Every claim references actual file paths, class/function names, and line numbers. Never write "the code is well-structured" without pointing to evidence.
- **Coverage**: Cover every major module. In large repositories, say which modules were sampled rather than implying exhaustive reading.
- **Accuracy**: Report only what is in the code. Do not infer features that are not present.
- **No secrets**: Name secret-bearing files and variables; never reproduce their values.
- **ASCII Diagrams**: Include ASCII flow diagrams in Phase 3.
- **Tables**: Use markdown tables for structured data (tech stack, entry points, components, integrations).
