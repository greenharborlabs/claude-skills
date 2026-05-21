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
| `*.py` generic | general-purpose | general-purpose | general-purpose | `pytest <pattern>` |
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
| Other stacks | skip | skip |

Spawn API contract reviewer only if aggregate diff touches:

- `*Controller.java`
- `*Resource.java`
- `*Dto.java`
- `*Request.java`
- `*Response.java`
- `*ControllerAdvice.java`
- `openapi.*`

Performance and concurrency concerns remain in the standard per-task reviewer.
