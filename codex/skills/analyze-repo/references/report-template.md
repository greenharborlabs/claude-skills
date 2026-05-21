# Repository Analysis Report: {PROJECT_NAME}

**Generated:** {DATE}
**Repository:** {REPO_PATH}

---

## PHASE 1: High-Level Overview

### Project Purpose
{What problem does this project solve? Who is the intended audience?}

### Tech Stack
| Category | Technology | Version |
|----------|-----------|---------|
| Language | | |
| Framework | | |
| Libraries | | |
| Build Tools | | |
| Database | | |
| Testing | | |

### Architecture Style
{e.g., monolith, microservices, agent-based, event-driven, etc. Justify with evidence.}

### Entry Points
| Entry Point | File | Purpose |
|-------------|------|---------|
| | | |

### Directory Structure
```
project-root/
├── dir/          # Description
│   ├── file      # Description
│   └── ...
└── ...
```

---

## PHASE 2: Core Components Deep Dive

### Component: {Name}
- **Location:** `path/to/files`
- **Responsibility:** {What does it do?}
- **Key Classes / Functions / Interfaces:**
  - `ClassName` (file:line) - {brief explanation}
  - `functionName` (file:line) - {brief explanation}
- **Internal Dependencies:** {modules it depends on}
- **External Dependencies:** {packages/libraries it uses}
- **Design Patterns:** {e.g., Factory, Strategy, Observer}

{Repeat for each major component}

---

## PHASE 3: Data Flow & Execution Flow

### Application Startup
```
Step 1: ...
Step 2: ...
```

### Key Operation Trace: {operation name}
```
[Input] → [Component A] → [Component B] → [Output]
```

### Data Transformations
{How data changes as it flows through the system}

### Async / Concurrent Patterns
{Event loops, message queues, worker threads, etc.}

---

## PHASE 4: Configuration & Integration Points

### Configuration
| Method | Location | Purpose |
|--------|----------|---------|
| env vars | `.env` | |
| config file | | |
| CLI args | | |

### External Integrations
| Service | Purpose | Auth Method |
|---------|---------|-------------|
| | | |

### Plugin / Extension Mechanisms
{How new integrations or features plug in}

---

## PHASE 5: AI / LLM-Specific Analysis

### LLM Usage
{Does the project use LLMs or AI agents? How?}

### Model Providers
| Provider | API | Model(s) |
|----------|-----|----------|
| | | |

### Prompt Management
{How prompts are structured, stored, and managed}

### Agent Orchestration
{Tool-use patterns, agent loops, multi-agent coordination}

### Context / Memory Management
{How context is maintained across interactions}

---

## PHASE 6: Code Quality & Architecture Assessment

### Strengths
- {Well-designed aspect with file/line reference}

### Weaknesses / Technical Debt
- {Fragile, incomplete, or overly complex area with reference}

### Security Concerns
- {Hardcoded secrets, unsafe inputs, insecure patterns}

### Test Coverage
| Area | Coverage | Notes |
|------|----------|-------|
| | | |

### Documentation Quality
{README quality, inline comments, API docs, self-documenting code}

---

## PHASE 7: How to Run & Extend

### Setup & Run
1. Prerequisites: ...
2. Install: ...
3. Configure: ...
4. Run: ...

### Adding a New Feature / Module
1. ...
2. ...

### Key Files for Onboarding
| Priority | File | Why |
|----------|------|-----|
| 1 | | |
| 2 | | |
| 3 | | |

---

## PHASE 8: Summary & Key Insights

### Architectural Decisions
- {Decision 1}
- {Decision 2}
- {Decision 3}

### Most Interesting / Novel Aspect
{The single most interesting or novel thing about this codebase}

### Recommended Next Steps
- {If contributing to or forking this project}
