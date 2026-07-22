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
- Methods/functions whose mixed responsibilities or cognitive load demonstrably obscure behavior
- Branching or nesting that makes reachable states difficult to reason about or test
- Loops with control flow that hides termination, skips required work, or creates a credible correctness/performance risk
- Classes or methods combining unrelated responsibilities with a concrete maintenance or testability cost
- Parameter lists that repeatedly cause call-site mistakes or expose a missing domain concept

#### Duplication
- Repeated behavior that is already drifting or creates a credible multi-site maintenance risk
- Copy-paste variants whose differences are accidental or hard to keep consistent
- Repeated identifiers whose coupling is semantic rather than merely textual
- Similar branch structures only when a shared policy or abstraction would reduce real inconsistency

#### Naming & Readability
- Names that materially misrepresent behavior, units, ownership, or lifecycle
- Abbreviations or short names that make non-local code meaningfully harder to understand
- Inconsistency that can cause incorrect usage or contract confusion; omit preference-only naming comments

#### Method & Class Structure
- Classes with multiple unrelated responsibilities and a demonstrated reason to change independently
- Utility classes accumulating unrelated behavior with unclear ownership
- Mixed abstraction levels or misplaced behavior only when they obscure the flow or deepen coupling

#### Error Handling Hygiene
- Empty catch blocks silently swallowing exceptions
- Catching overly broad exception types when specific types are appropriate
- Logging an exception and then re-throwing it (double logging)
- Returning `null` when it violates the established contract or creates an unchecked failure path
- Missing try-with-resources for closeable resources
- Logging only `e.getMessage()` instead of the full exception

#### Test Gaps
- New code paths without corresponding test coverage
- Tests asserting on type/status but not side effects
- Missing negative-path assertions
- Tests with no assertions
- Test methods testing multiple unrelated behaviors

#### Magic Numbers & String Coupling
- Unexplained literals whose meaning, unit, or policy is unclear and reused or safety-relevant
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
