---
name: analyze-repo
description: Perform a comprehensive, multi-phase technical analysis of a git repository. This skill should be used when the user wants to understand a codebase deeply, analyze repository architecture, generate a technical report, or says 'analyze-repo'. It launches parallel sub-agents to examine project structure, components, data flow, configuration, AI patterns, code quality, and provides onboarding guidance. Triggers on "analyze repo", "analyze this repo", "analyze codebase", "repo analysis", "codebase analysis", "technical report".
---

# Analyze Repo

## Overview

Generate a comprehensive technical analysis report for the current git repository. The analysis covers 8 phases: high-level overview, core components deep dive, data flow analysis, configuration & integrations, AI/LLM patterns, code quality assessment, setup & extension guide, and summary insights.

## Help

If the argument is `--help` or `-h`, display the following help text and stop (do not run the analysis):

```
analyze-repo — Comprehensive multi-phase git repository analysis

USAGE
  /analyze-repo [OPTIONS] [OUTPUT_FILE]

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
  /analyze-repo                              Full analysis → analysis-report.md
  /analyze-repo report.md                    Full analysis → report.md
  /analyze-repo --quick                      Skip data-flow tracing for speed
  /analyze-repo --phases 1,2,6               Only run overview, components, and quality phases
  /analyze-repo --phases 1,6 quick-check.md  Partial analysis to a custom file
```

## Arguments

Accept optional arguments:

| Argument | Description | Default |
|----------|-------------|---------|
| `OUTPUT_FILE` | Path for the generated report | `analysis-report.md` |
| `--help`, `-h` | Show help text and exit | — |
| `--phases LIST` | Comma-separated phase numbers (1-8) to include | all phases |
| `--quick` | Skip Phase 3 (data-flow tracing) for faster results | off |

Parse arguments in this order: flags first (`--help`, `--quick`, `--phases`), then any remaining positional argument is the output file path.

Example invocations:
- `/analyze-repo` — full analysis → `analysis-report.md`
- `/analyze-repo my-project-analysis.md` — full analysis → `my-project-analysis.md`
- `/analyze-repo --quick` — skip data-flow tracing → `analysis-report.md`
- `/analyze-repo --phases 1,2,6 report.md` — only phases 1, 2, 6 → `report.md`

## Workflow

### Step 0: Preparation

1. If `--help` or `-h` is present, display the help text above and stop.
2. Parse flags: `--quick`, `--phases LIST`. Remaining positional arg is the output file path (default: `analysis-report.md`).
3. Confirm the current directory is a git repository by checking for `.git/`.
4. Capture the repository name from the directory name or `git remote get-url origin`.
5. Read `references/report-template.md` from this skill's directory to understand the report structure.
6. Determine which agents to launch based on `--phases` and `--quick` flags (see mapping below).

**Phase → Agent mapping:**
| Phases needed | Agent to launch |
|---------------|-----------------|
| 1 | Agent 1 (Structure & Stack) |
| 2 | Agent 2 (Core Components) |
| 3 | Agent 5 (Data Flow) — skipped if `--quick` |
| 4, 5 | Agent 3 (Config, Integration & AI) |
| 6, 7 | Agent 4 (Quality, Tests & Docs) |
| 8 | No agent — synthesized from other phases |

### Step 1: Parallel Discovery (Launch 4 Explore Agents)

Launch these 4 agents concurrently using the Agent tool with `subagent_type: "Explore"` and thoroughness level "very thorough":

**Agent 1 — Structure & Stack Discovery:**
> Explore the repository to determine: (a) project purpose from README, CLAUDE.md, package.json, or equivalent; (b) complete tech stack with versions from package.json, requirements.txt, Cargo.toml, go.mod, pom.xml, or equivalent dependency files; (c) architecture style based on directory structure and module organization; (d) all entry points (main files, scripts, CLI commands, server start); (e) annotated directory tree of top-level structure. Report findings with specific file paths and line numbers.

**Agent 2 — Core Components Inventory:**
> Explore every major module/component in the repository. For each, report: (a) name and file path(s); (b) responsibility; (c) key classes, functions, interfaces with file:line references; (d) internal and external dependencies; (e) design patterns used (Factory, Strategy, Observer, etc.). Be exhaustive — cover all source directories. Report findings with specific file paths and line numbers.

**Agent 3 — Configuration, Integration & AI Analysis:**
> Explore the repository for: (a) all configuration mechanisms (env vars, config files, CLI args) with file locations; (b) external API/service/database integrations and how they authenticate; (c) plugin or extension mechanisms; (d) any LLM/AI agent usage — identify model providers, prompt structures, agent orchestration patterns, tool-use patterns, and context/memory management. Report findings with specific file paths and line numbers.

**Agent 4 — Quality, Tests & Documentation Assessment:**
> Explore the repository for: (a) code quality strengths — well-designed patterns, elegant implementations; (b) weaknesses and technical debt — fragile, incomplete, or overly complex areas; (c) security concerns — hardcoded secrets, unsafe inputs, insecure patterns; (d) test coverage — test files, what is covered, what is missing; (e) documentation quality — README completeness, inline comments, API docs. Also determine: (f) how to set up and run the project locally; (g) how a developer would add a new feature; (h) the most important files for onboarding. Report findings with specific file paths and line numbers.

### Step 2: Data Flow Analysis (Launch 1 Agent)

**Skip this step if `--quick` is set or Phase 3 is excluded by `--phases`.**

After Step 1 completes, launch one more agent to trace execution flow. Use the Agent tool with `subagent_type: "Explore"` and thoroughness "very thorough":

**Agent 5 — Execution & Data Flow Tracing:**
> Trace the execution path of this application from startup to a key operation. (a) Map the startup sequence — what happens when the application starts, in order; (b) trace a key user-facing operation end-to-end through the codebase; (c) identify how data is transformed as it moves between components; (d) note all async, concurrent, or event-driven patterns (event emitters, message queues, callbacks, promises, streams). Create ASCII diagrams showing the flow. Report findings with specific file paths and line numbers.

This agent benefits from knowledge of the components discovered in Step 1, so run it after Step 1 completes.

### Step 3: Synthesize Report

Once all agents have returned their findings:

1. Read `references/report-template.md` from this skill's directory for the report structure.
2. Synthesize all agent findings into the 8-phase report format.
3. Replace all template placeholders with actual data.
4. Ensure every observation references specific file names, function names, and line numbers.
5. Remove any sections that are not applicable (e.g., Phase 5 if no AI/LLM usage found) with a brief note explaining why.

### Step 4: Write Report

Write the completed report to the output file path determined in Step 0.

## Quality Standards

- **Specificity**: Every claim must reference actual file paths, class/function names, and line numbers. Never use vague language like "the code is well-structured" without pointing to specific evidence.
- **Completeness**: Cover all source files. Do not skip modules or components.
- **Accuracy**: Only report what is actually in the code. Do not assume or infer features that are not present.
- **ASCII Diagrams**: Include ASCII flow diagrams in Phase 3 to visualize execution and data flow.
- **Tables**: Use markdown tables for structured data (tech stack, entry points, components, integrations).

## Resources

### references/
- `report-template.md` — The markdown template structure for the output report. Read this to understand the expected format and fill in all sections.
