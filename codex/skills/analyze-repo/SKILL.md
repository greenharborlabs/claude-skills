---
name: analyze-repo
description: Perform a comprehensive multi-phase technical analysis of a git repository. Use when the user wants to understand a codebase deeply, analyze repository architecture, generate a technical report, or says "analyze-repo", "analyze this repo", "analyze codebase", "repo analysis", or "technical report".
---

# Analyze Repo

Generate a comprehensive technical analysis report for the current git repository. Cover repository purpose, tech stack, core components, data flow, configuration and integrations, AI/LLM patterns, code quality, setup, extension guidance, and summary insights.

## Help

If the argument is `--help` or `-h`, show:

```text
analyze-repo - Comprehensive multi-phase git repository analysis

Usage:
  analyze-repo [OPTIONS] [OUTPUT_FILE]

Arguments:
  OUTPUT_FILE        Path for the generated report (default: analysis-report.md)

Options:
  --help, -h         Show this help
  --phases LIST      Comma-separated phase numbers to run, e.g. --phases 1,2,6
  --quick            Skip detailed data-flow tracing
```

## Arguments

- `OUTPUT_FILE`: default `analysis-report.md`.
- `--phases LIST`: include only phases 1-8 listed below.
- `--quick`: skip Phase 3 unless explicitly requested.

## Workflow

1. Confirm the current directory is a git repository.
2. Read `references/report-template.md` for the report structure.
3. Gather facts with read-only commands and file reads. Prefer `rg`, `rg --files`, `find`, `git log`, `git remote`, and manifest/config reads.
4. Run the selected phases yourself in order. If the user explicitly requested parallel agent work, distinct phase groups may be delegated to Codex explorer agents; otherwise do the analysis directly.
5. Synthesize findings into the report template.
6. Write the report to the requested output path.

## Phases

1. **High-Level Overview**: purpose, tech stack, architecture style, entry points, directory structure.
2. **Core Components**: per-module responsibility, key files/symbols, dependencies, design patterns.
3. **Data Flow & Execution**: startup sequence and one or more key operation traces with ASCII diagrams.
4. **Configuration**: env vars, config files, integrations, plugin mechanisms.
5. **AI / LLM Analysis**: providers, prompts, agent orchestration, tool-use patterns, memory/context handling.
6. **Code Quality**: strengths, tech debt, security concerns, tests, docs.
7. **How to Run & Extend**: setup instructions, extension guide, onboarding file list.
8. **Summary & Key Insights**: architectural decisions, notable aspects, recommended next steps.

## Quality Standards

- Every non-trivial claim must cite file paths and, when useful, line numbers or symbols.
- Report repo evidence only. Do not invent missing features or infer behavior not supported by code.
- Use markdown tables for structured facts and ASCII diagrams for flows.
- If a phase is not applicable, keep a short "Not applicable" note with evidence.
- Keep the report useful for onboarding: accurate entry points, run commands, and the files a new engineer should read first.

## Resources

- `references/report-template.md`: output report structure.
