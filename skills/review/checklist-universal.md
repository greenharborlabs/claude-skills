# Universal Pre-Landing Review Checklist

## Instructions

Review the `git diff origin/main` output for the issues listed below. Be specific — cite `file:line` and suggest fixes. Skip anything that's fine. Only flag real problems.

**Two-pass review:**
- **Pass 1 (CRITICAL):** Hardcoded secrets, race conditions, trust boundary violations. These block landing.
- **Pass 2 (INFORMATIONAL):** Dead code, test gaps, magic numbers, stale comments, crypto misuse, version consistency.

**Output format:**

```
Pre-Landing Review: N issues (X critical, Y informational)

**CRITICAL** (blocking):
- [file:line] Problem description
  Fix: suggested fix

**Issues** (non-blocking):
- [file:line] Problem description
  Fix: suggested fix
```

If no issues found: `Pre-Landing Review: No issues found.`

Be terse. For each issue: one line describing the problem, one line with the fix. No preamble, no summaries, no "looks good overall."

---

## Review Categories

### Pass 1 — CRITICAL

#### Hardcoded Secrets & Credentials
- API keys, tokens, passwords, or connection strings committed in source (including test fixtures that look like real credentials)
- `.env` files or credential configs added to version control without `.gitignore` coverage
- Private keys, certificates, or signing secrets in the diff

#### Race Conditions
- Check-then-act patterns without atomicity (read a value, branch on it, then write — without a lock, transaction, or atomic operation)
- `find_or_create` patterns without unique constraints or conflict handling
- Status transitions that don't use atomic compare-and-swap

#### Trust Boundary Violations
- Untrusted input (user input, LLM output, external API responses) used directly in security-sensitive operations without validation
- User-controlled values in SQL, shell commands, file paths, or redirect URLs without sanitization
- LLM-generated content written to DB or rendered in UI without format/type validation

### Pass 2 — INFORMATIONAL

#### Dead Code & Unused Declarations
- Variables assigned but never read
- Imports/requires that are unused
- Functions/methods defined but never called within the diff scope
- Parameters declared but never referenced in the method body
- **Constant/redundant parameters**: method parameters that receive the same value at every call site (same field, same literal, or always equal to another parameter) — the parameter is dead flexibility. Either inline the constant value inside the method, or it's a design issue where callers should be passing different values but aren't.
- Private fields/methods that are never accessed
- Unreachable code after `return`, `throw`, `break`, or `continue`
- Dead branches: `if` conditions that are always true/false based on preceding logic
- Empty `catch`, `then`, or `finally` blocks that silently swallow errors

#### Complexity
- Methods/functions exceeding ~20 lines — candidates for extraction
- Cyclomatic complexity: methods with more than 4-5 branching paths (`if`/`else`/`switch`/`catch`/`&&`/`||`/ternary) — extract helper methods or use early returns
- Cognitive complexity: deeply nested logic (3+ levels of `if`/`for`/`while` nesting) — flatten with guard clauses or extract inner blocks
- God methods: single methods doing more than one conceptual thing (e.g., validate + transform + persist) — split into focused methods
- Long parameter lists (5+ parameters) — consider a parameter object or builder

#### Duplication
- Repeated code blocks (3+ lines) that appear in multiple places — extract to a shared method or utility
- Copy-paste patterns where logic is identical except for one variable — parameterize
- Repeated string literals used as identifiers or keys in multiple locations — extract to constants
- Similar `switch`/`if-else` chains in multiple methods — consider polymorphism or a strategy map

#### Naming & Readability
- Boolean variables or methods without question-word prefixes (`is`, `has`, `should`, `can`) — `valid` → `isValid`
- Abbreviated or single-letter variable names outside of trivial loop indices — `p` → `payment`, `svc` → `service`
- Methods whose names don't describe what they do (e.g., `process()`, `handle()`, `doStuff()` without qualification)
- Inconsistent naming conventions within the same file or package (mixing `camelCase` and `snake_case`, or `get`/`fetch`/`retrieve` for the same pattern)

#### Method & Class Structure
- Classes with too many responsibilities (doing unrelated things) — candidate for splitting
- Utility/helper classes that are dumping grounds for unrelated static methods
- Methods that mix different levels of abstraction (high-level orchestration alongside low-level string manipulation)
- Feature envy: methods that primarily operate on another class's data — consider moving the method

#### Error Handling Hygiene
- Empty catch blocks that silently swallow exceptions
- Catching overly broad exception types (`Exception`, `Error`, `Throwable`) when specific types are appropriate
- Logging an exception and then re-throwing it (double logging)
- Returning `null` from a method where a failure should throw or return an `Optional`/`Result`
- Missing `finally` or try-with-resources for closeable resources (streams, connections, files)
- Logging only `e.getMessage()` instead of the full exception — loses exception type, stack trace, and chained causes. Pass the exception object itself to the logger.

#### Test Gaps
- New code paths without corresponding test coverage
- Tests that assert on type/status but not side effects (was the record actually created? was the callback fired?)
- Missing negative-path assertions (what happens when input is invalid?)
- Tests with no assertions (test methods that call code but never verify behavior)
- Test methods testing multiple unrelated behaviors — split into focused test methods

#### Magic Numbers & String Coupling
- Bare numeric literals used in logic without named constants
- Error message strings matched elsewhere as query filters (grep the string — is anything depending on its exact text?)
- Hardcoded URLs, ports, or hostnames that should come from configuration

#### Stale Comments
- Comments or docstrings that describe old behavior after the code changed in this diff
- TODO comments referencing completed work
- Commented-out code blocks (dead code hidden in comments) — remove or restore

#### Crypto & Entropy Misuse
- `rand()` / `Math.random()` / `Random` for security-sensitive values (tokens, secrets, nonces) — use `SecureRandom` or `crypto.randomBytes` instead
- Non-constant-time comparisons (`==`, `===`) on secrets or tokens — vulnerable to timing attacks
- Truncation of data instead of hashing (last N chars instead of SHA-256)

#### Version & Changelog Consistency
- Version mismatch between package.json/build.gradle/pom.xml and CHANGELOG/release notes
- CHANGELOG entries that describe changes inaccurately

---

## Suppressions — DO NOT flag these

- "X is redundant with Y" when the redundancy is harmless and aids readability
- "Add a comment explaining why this threshold/constant was chosen" — thresholds change during tuning, comments rot
- "This assertion could be tighter" when the assertion already covers the behavior
- Suggesting consistency-only changes with no functional impact
- Harmless no-ops
- ANYTHING already addressed in the diff you're reviewing — read the FULL diff before commenting
