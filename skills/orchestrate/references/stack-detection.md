# Stack Detection

Load this when resolving agent types, test commands, and specialists.

## Agent Suite

Use explicit `--coder`, `--reviewer`, or `--architect` values when provided.
Otherwise scan the plan file's `**Files:**` sections plus a quick project-root glob.

| Signal | Coder | Reviewer | Architect | Test Command |
| --- | --- | --- | --- | --- |
| `*.java` + `build.gradle*` | `backend-coder-java` | `backend-reviewer-java` | `backend-planning-architect` | `./gradlew test --tests "<pattern>"` |
| `*.java` + `pom.xml` | `backend-coder-java` | `backend-reviewer-java` | `backend-planning-architect` | `mvn -pl . test -Dtest="<pattern>"` |
| `CLAUDE.md` state markers + `nanoclaw`/`container` | `nanoclaw-coder` | `nanoclaw-reviewer` | `nanoclaw-architect` | `npx jest --testPathPattern="<pattern>"` |
| `*.py` + `openclaw`/`alpaca`/`kalshi` | `openclaw-coder` | `openclaw-reviewer` | `openclaw-architect` | `pytest <pattern>` |
| `*.ts`/`*.tsx` + `vitest` | `frontend-impl` | `frontend-reviewer` | `frontend-architect` | `npx vitest run <pattern>` |
| `*.ts`/`*.tsx` + `jest` no NanoClaw | `frontend-impl` | `frontend-reviewer` | `frontend-architect` | `npx jest --testPathPattern="<pattern>" --no-coverage` |
| `*.py` generic | `backend-coder-python` | `backend-reviewer-python` | `backend-planning-architect-python` | `pytest <pattern>` |
| Mixed/unknown | ask with detected signals | ask | ask | ask |

For Gradle multi-project builds, prefix the module, e.g.
`./gradlew :api:test --tests "<pattern>"`. For Maven multi-module, use `-pl`.
Determine the module from file paths.

Resolve `TEST_CMD` to a literal command in every Coder prompt. Sub-agents cannot
dereference orchestrator-local variables.

Print:

```text
Agent suite: <coder> / <reviewer> / <architect> (detected from: <signal>)
Test command: <TEST_CMD>
```

## Specialist Reviewers

| Signal | Security Reviewer | API Contract Reviewer |
| --- | --- | --- |
| Java/Gradle/Maven | `backend-security-reviewer-java` | `backend-api-contract-reviewer-java` only if API files changed |
| Python | `backend-security-reviewer-python` only if security-sensitive files changed | `backend-api-contract-reviewer-python` only if API/CLI contract files changed |
| Other stacks | skip | skip |

For Java, spawn API contract reviewer only if aggregate diff touches:

- `*Controller.java`
- `*Resource.java`
- `*Dto.java`
- `*Request.java`
- `*Response.java`
- `*ControllerAdvice.java`
- `openapi.*`

For Python, spawn security reviewer if aggregate diff touches files or code paths
with security-sensitive names or content:

- `*auth*.py`, `*security*.py`, `*credential*.py`, `*secret*.py`
- `*payer*.py`, `*wallet*.py`, `*payment*.py`, `*token*.py`
- config/env handling, redaction, subprocess, filesystem state, or HTTP client code

For Python, spawn API contract reviewer if aggregate diff touches:

- FastAPI/Flask route modules, `api/**/*.py`, `routes/**/*.py`, `views/**/*.py`
- Typer/Click/argparse CLI modules such as `cli.py`
- public SDK exports in `__init__.py`
- request/response/schema/model files, error envelope code, or `openapi.*`

Performance and concurrency concerns remain in the standard per-task reviewer.
