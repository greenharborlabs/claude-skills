# Java / Spring Boot Pre-Landing Review Checklist

## Instructions

Review the `git diff origin/main` output for Java/Spring-specific issues. Be specific — cite `file:line` and suggest fixes. Skip anything that's fine. Only flag real problems.

---

## Review Categories

### Pass 1 — CRITICAL

#### Transaction Hazards
- `@Transactional` wrapping external HTTP calls, AI/LLM calls, or message sends — move external calls outside transaction
- `@Transactional` on private methods (silently ignored by Spring proxies)

#### N+1 Queries
- Lazy-loaded associations accessed in loops without `JOIN FETCH` or `@EntityGraph`
- Repository methods returning entities whose associations are accessed without eager fetching

#### SQL Injection
- String concatenation in `@Query` native queries — use parameterized `:name` placeholders
- `EntityManager.createNativeQuery()` with string interpolation

#### Input Validation
- Missing `@Valid` or `@Validated` on `@RequestBody` / `@RequestParam` at controller boundaries
- Missing `@NotNull` / `@NotBlank` on DTO fields that must not be null

#### Broken Object-Level Authorization (BOLA)
- User-supplied IDs used to fetch records without ownership/authorization check
- Missing `@PreAuthorize` or manual authorization on endpoints modifying user-specific data

#### Missing Authorization
- Controller endpoints without any authorization annotation or programmatic check
- New endpoints not covered by `SecurityFilterChain` rules

### Pass 2 — INFORMATIONAL

#### Transaction Efficiency
- Missing `readOnly = true` on `@Transactional` for read-only operations
- Overly broad transaction scope

#### Dependency Injection
- Field injection (`@Autowired` on fields) instead of constructor injection
- Missing `final` on constructor-injected fields

#### HTTP Client Resilience
- `RestTemplate` / `WebClient` calls without explicit timeouts
- Missing retry or circuit breaker on external service calls

#### Pagination
- Repository methods returning `List<Entity>` for unbounded result sets — use `Page` or `Slice`
- Missing default/max page size limits

#### Virtual Thread Compatibility
- `synchronized` blocks in code that may run on Virtual Threads
- Thread-local storage assumptions in Virtual Thread context

#### Error Handling
- Missing `@ControllerAdvice` / `@ExceptionHandler` for custom exception types
- Catching generic `Exception` instead of specific types

#### Code Quality — Java-Specific

**Unused & Dead Code:**
- Unused fields, constants, imports, or local variables
- Private methods never called
- Unused method parameters
- **Constant/redundant parameters**: trace every call site — if always same value, inline it
- Empty method bodies
- Unused `@SuppressWarnings`

**Constructor & Dependency Bloat:**
- Constructor parameters assigned to fields but field never read — remove
- Fields with setters but no reads — dead state
- Injected dependencies never called — remove
- Constructor parameters derivable from already-injected config — derive instead

**Complexity & Structure:**
- Methods exceeding ~20 lines
- `if`/`else` chains with 4+ branches — consider switch/pattern matching/strategy
- Nested if depth > 3 — flatten with guard clauses
- Classes with 10+ methods or 300+ lines
- `switch` without `default` (unless exhaustive sealed type)

**Modern Java (Java 17+):**
- Mutable DTOs that could be records
- `instanceof` chains → pattern matching
- Sealed interface opportunities
- Text blocks for multiline strings
- `Optional.get()` without `.isPresent()` check
- `.stream().collect(Collectors.toList())` → `.stream().toList()`

**Null Safety:**
- Methods returning `null` where `Optional` is clearer
- Dereferencing potentially null values without check
- Primitive wrapper types where primitives suffice

**Immutability & Thread Safety:**
- Mutable fields in multi-threaded classes without synchronization
- Returning mutable internal collections directly
- `Date`/`Calendar` instead of `java.time`

**Logging:**
- String concatenation in log statements — use parameterized logging
- Logging sensitive data
- Logging only `e.getMessage()` — pass the exception object itself

**Resource Management:**
- Streams, connections not in try-with-resources
- Connection pools without proper configuration

**Spring-Specific Smells:**
- `@Component`/`@Service` on classes that should be plain POJOs
- `@Value` injection of complex expressions — use `@ConfigurationProperties`
- Hardcoded `@Scheduled` cron expressions

---

## Suppressions — DO NOT flag these

- Threshold/constant value choices
- "Add Javadoc" suggestions
- Style-only changes
- Test utility patterns that work correctly
- Lombok-generated code smells
- Framework-required patterns (e.g., empty no-arg constructors for JPA)
- ANYTHING already addressed in the diff
