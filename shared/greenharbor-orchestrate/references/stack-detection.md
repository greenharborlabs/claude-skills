# Stack Detection

Load this when resolving agent types, test commands, and specialists.

## Agent Suite

Use explicit `--coder`, `--reviewer`, or `--architect` values when provided.
Otherwise scan the plan file's `**Files:**` sections and `## Risk Flags` plus a
quick project-root glob.

| Signal | Coder | Reviewer | Architect | Test Command |
| --- | --- | --- | --- | --- |
| `*.java` + `build.gradle*` | `greenharbor-backend-coder-java` | `greenharbor-backend-reviewer-java` | `greenharbor-backend-planning-architect` | `./gradlew test --tests "<pattern>"` |
| `*.java` + `pom.xml` | `greenharbor-backend-coder-java` | `greenharbor-backend-reviewer-java` | `greenharbor-backend-planning-architect` | `<maven-command> -pl . test -Dtest="<pattern>"` |
| `Cargo.toml` + `*.rs` | `greenharbor-backend-coder-rust` | `greenharbor-backend-reviewer-rust` | `greenharbor-backend-planning-architect-rust` | `cargo test -p "<package>" "<pattern>"` |
| `CLAUDE.md` state markers + `nanoclaw`/`container` | `greenharbor-nanoclaw-coder` | `greenharbor-nanoclaw-reviewer` | `greenharbor-nanoclaw-architect` | `npx jest --testPathPattern="<pattern>"` |
| `*.py` + `openclaw`/`alpaca`/`kalshi` | `greenharbor-openclaw-coder` | `greenharbor-openclaw-reviewer` | `greenharbor-openclaw-architect` | `pytest <pattern>` |
| `*.ts`/`*.tsx` + `vitest` | `greenharbor-frontend-impl` | `greenharbor-frontend-reviewer` | `greenharbor-frontend-architect` | `npx vitest run <pattern>` |
| `*.ts`/`*.tsx` + `jest` no NanoClaw | `greenharbor-frontend-impl` | `greenharbor-frontend-reviewer` | `greenharbor-frontend-architect` | `npx jest --testPathPattern="<pattern>" --no-coverage` |
| `*.py` generic | `greenharbor-backend-coder-python` | `greenharbor-backend-reviewer-python` | `greenharbor-backend-planning-architect-python` | `pytest <pattern>` |
| Mixed/unknown | ask with detected signals | ask | ask | ask |

For Gradle multi-project builds, prefix the module, e.g.
`./gradlew :api:test --tests "<pattern>"`. For Maven multi-module, use `-pl`.
Resolve `<maven-command>` to `./mvnw` when present; otherwise use the repository-
or CI-documented Maven command. Determine the module from file paths.

Resolve `TEST_CMD` to a literal command in every Coder prompt. Sub-agents cannot
dereference orchestrator-local variables.

Print:

```text
Agent suite: <coder> / <reviewer> / <architect> (detected from: <signal>)
Test command: <TEST_CMD>
```

## Specialist Reviewers

| Signal | Security Reviewer | API Contract Reviewer | Performance/Concurrency Lens |
| --- | --- | --- | --- |
| Java/Gradle/Maven | `greenharbor-backend-security-reviewer-java` if security risk | `greenharbor-backend-api-contract-reviewer-java` if API files or public-api risk | standard `greenharbor-backend-reviewer-java` covers performance/concurrency |
| Python | `greenharbor-backend-security-reviewer-python` if security risk or sensitive files | `greenharbor-backend-api-contract-reviewer-python` if API/CLI files or public-api risk | standard `greenharbor-backend-reviewer-python` covers performance/concurrency |
| Rust/Cargo | `greenharbor-backend-security-reviewer-rust` if security risk or sensitive files | `greenharbor-backend-api-contract-reviewer-rust` if public API, HTTP/CLI/FFI, or schema files change | standard `greenharbor-backend-reviewer-rust` covers performance/concurrency |
| React/TypeScript | `greenharbor-frontend-reviewer` if security risk | `greenharbor-frontend-reviewer` if public-api risk | `greenharbor-frontend-reviewer` if performance/concurrency risk |
| Other stacks | skip | skip | standard reviewer only |

For Java, spawn security reviewer when the plan marks `security: yes` or the
aggregate diff touches auth/security/credential/secret/token handling. Spawn API
contract reviewer when the plan marks `public-api: yes` or aggregate diff touches:

- `*Controller.java`
- `*Resource.java`
- `*Dto.java`
- `*Request.java`
- `*Response.java`
- `*ControllerAdvice.java`
- `openapi.*`

For Python, spawn security reviewer if the plan marks `security: yes` or
aggregate diff touches files or code paths with security-sensitive names or
content:

- `*auth*.py`, `*security*.py`, `*credential*.py`, `*secret*.py`
- `*payer*.py`, `*wallet*.py`, `*payment*.py`, `*token*.py`
- config/env handling, redaction, subprocess, filesystem state, or HTTP client code

For Python, spawn API contract reviewer if the plan marks `public-api: yes` or
aggregate diff touches:

- FastAPI/Flask route modules, `api/**/*.py`, `routes/**/*.py`, `views/**/*.py`
- Typer/Click/argparse CLI modules such as `cli.py`
- public SDK exports in `__init__.py`
- request/response/schema/model files, error envelope code, or `openapi.*`

For Rust, resolve the package with `cargo metadata --no-deps` when the plan
touches a workspace member. Use `cargo test "<pattern>"` for a single package
when no explicit package selection is required. If the plan spans Rust and
another stack, treat it as mixed/unknown unless an explicit agent override or
the plan scope isolates one stack.

Spawn the Rust security reviewer when the plan marks `security: yes` or the
aggregate diff touches authentication, authorization, credentials, secrets,
tokens, cryptography, outbound HTTP, filesystem/process boundaries, `unsafe`,
`extern`, FFI, `build.rs`, registry configuration, or security-sensitive
dependency-source changes.

Spawn the Rust API contract reviewer when the plan marks `public-api: yes` or
the aggregate diff touches:

- crate roots or public exports (`src/lib.rs`, public modules, changed `pub`
  items, public error/types/traits)
- route/handler, request/response/schema, serde, OpenAPI, or error-envelope code
- CLI command/argument/config/output definitions
- FFI exports, ABI types, or feature/MSRV declarations

For React/TypeScript, use `greenharbor-frontend-reviewer` as a targeted specialist when
`security`, `public-api`, `performance`, or `concurrency` risk is `yes`, because
that reviewer covers XSS/CSP, React contracts, accessibility, and performance.
