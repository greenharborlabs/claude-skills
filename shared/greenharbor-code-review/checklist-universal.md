# Universal Pre-Landing Review Checklist

## Instructions

Apply these checks to the active review input and use the output format from `SKILL.md`.
Only flag real problems with specific `file:line` evidence. Skip anything that is fine or already addressed.

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
- Cyclomatic complexity: methods with more than 4-5 branching paths — extract helper methods or use early returns
- Cognitive complexity: deeply nested logic (3+ levels) — flatten with guard clauses or extract inner blocks
- Multi-break/multi-continue loops — extract to method with early returns or use streams
- Nested loops — extract inner loop to named methods
- God methods: single methods doing more than one conceptual thing — split into focused methods
- Long parameter lists (5+ parameters) — consider a parameter object or builder

#### Duplication
- Repeated code blocks (3+ lines) in multiple places — extract to shared method
- Copy-paste patterns differing in one variable — parameterize
- Repeated string literals as identifiers — extract to constants
- Similar switch/if-else chains — consider polymorphism or strategy map

#### Naming & Readability
- Boolean variables without question-word prefixes (`is`, `has`, `should`, `can`)
- Abbreviated or single-letter variable names outside trivial loop indices
- Methods whose names don't describe what they do
- Inconsistent naming conventions within the same file/package

#### Method & Class Structure
- Classes with too many responsibilities — candidate for splitting
- Utility classes as dumping grounds for unrelated static methods
- Methods mixing different abstraction levels
- Feature envy: methods primarily operating on another class's data

#### Error Handling Hygiene
- Empty catch blocks silently swallowing exceptions
- Catching overly broad exception types when specific types are appropriate
- Logging an exception and then re-throwing it (double logging)
- Returning `null` where failure should throw or return Optional/Result
- Missing try-with-resources for closeable resources
- Logging only `e.getMessage()` instead of the full exception

#### Test Gaps
- New code paths without corresponding test coverage
- Tests asserting on type/status but not side effects
- Missing negative-path assertions
- Tests with no assertions
- Test methods testing multiple unrelated behaviors

#### Magic Numbers & String Coupling
- Bare numeric literals without named constants
- Error message strings matched elsewhere as query filters
- Hardcoded URLs, ports, or hostnames that should be configuration

#### Stale Comments
- Comments describing old behavior after code changed
- TODO comments referencing completed work
- Commented-out code blocks — remove or restore

#### Crypto & Entropy Misuse
- `rand()`/`Math.random()`/`Random` for security-sensitive values — use SecureRandom or crypto.randomBytes
- Non-constant-time comparisons on secrets/tokens
- Truncation instead of hashing

#### Version & Changelog Consistency
- Version mismatch between build files and CHANGELOG
- CHANGELOG entries that describe changes inaccurately

---

## Suppressions — DO NOT flag these

- Redundancy that aids readability
- "Add a comment explaining threshold" suggestions
- "This assertion could be tighter" when it already covers behavior
- Consistency-only changes with no functional impact
- Harmless no-ops
- ANYTHING already addressed in the diff
