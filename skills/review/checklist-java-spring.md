# Java / Spring Boot Pre-Landing Review Checklist

## Instructions

Review the `git diff origin/main` output for Java/Spring-specific issues. Be specific — cite `file:line` and suggest fixes. Skip anything that's fine. Only flag real problems.

---

## Review Categories

### Pass 1 — CRITICAL

#### Transaction Hazards
- `@Transactional` wrapping external HTTP calls, AI/LLM calls, or message sends — the transaction holds a DB connection while waiting on the network. Move external calls outside the transaction boundary.
- `@Transactional` on private methods (silently ignored by Spring proxies)

#### N+1 Queries
- Lazy-loaded associations accessed in loops without `JOIN FETCH` or `@EntityGraph` — add fetch join or entity graph annotation
- Repository methods returning entities whose associations are accessed in service/controller layer without eager fetching

#### SQL Injection
- String concatenation in `@Query` native queries — use parameterized `:name` placeholders
- `EntityManager.createNativeQuery()` with string interpolation

#### Input Validation
- Missing `@Valid` or `@Validated` on `@RequestBody` / `@RequestParam` at controller boundaries
- Missing `@NotNull` / `@NotBlank` on DTO fields that must not be null

#### Broken Object-Level Authorization (BOLA)
- User-supplied IDs (path variables, request params) used to fetch records without ownership/authorization check
- Missing `@PreAuthorize` or manual authorization on endpoints that modify user-specific data

#### Missing Authorization
- Controller endpoints without any authorization annotation (`@PreAuthorize`, `@Secured`, `@RolesAllowed`) or programmatic check
- New endpoints not covered by `SecurityFilterChain` rules

### Pass 2 — INFORMATIONAL

#### Transaction Efficiency
- Missing `readOnly = true` on `@Transactional` for read-only operations
- Overly broad transaction scope (entire method transactional when only a subset needs it)

#### Dependency Injection
- Field injection (`@Autowired` on fields) instead of constructor injection
- Missing `final` on constructor-injected fields

#### HTTP Client Resilience
- `RestTemplate` / `WebClient` calls without explicit connect/read timeouts
- Missing retry or circuit breaker on external service calls

#### Pagination
- Repository methods returning `List<Entity>` for potentially unbounded result sets — use `Page<Entity>` or `Slice<Entity>` with `Pageable`
- Missing default/max page size limits

#### Virtual Thread Compatibility
- `synchronized` blocks or `ReentrantLock` in code that may run on Virtual Threads — use `Lock` implementations or restructure
- Thread-local storage assumptions in Virtual Thread context

#### Error Handling
- Missing `@ControllerAdvice` / `@ExceptionHandler` for custom exception types
- Catching generic `Exception` instead of specific types

#### Code Quality — Java-Specific (Sonar-equivalent)

**Unused & Dead Code:**
- Unused fields, constants, imports, or local variables
- Private methods never called from within the class
- Unused method parameters (especially in non-interface implementations)
- Empty method bodies (other than intentional no-ops in abstract/template patterns)
- Unused `@SuppressWarnings` annotations

**Constructor & Dependency Bloat:**
- Constructor parameters that are assigned to fields but the field is never read anywhere in the class — remove the field and parameter
- Fields with setters but no getters or internal reads — the field is dead state, remove it
- Injected dependencies (constructor params in `@Service`/`@Component` classes) that are never called — remove the dependency
- Constructor parameters that could be derived internally (e.g., from an already-injected config/properties object) — derive instead of injecting separately
- For each constructor parameter: trace whether the corresponding field is actually used in any method body. If not, it's dead weight.

**Complexity & Structure:**
- Methods exceeding ~20 lines — extract into focused private methods
- `if`/`else` chains with 4+ branches — consider `switch` (with pattern matching in Java 21+), strategy pattern, or a map-based dispatch
- Nested `if` depth > 3 — flatten with guard clauses (early returns)
- Classes with 10+ methods or 300+ lines — candidate for splitting by responsibility
- `switch` without `default` (unless exhaustive sealed type)
- Long constructor parameter lists — consider builder pattern

**Modern Java (Java 17+):**
- Mutable DTOs / data carriers that could be `record` types
- `instanceof` chains that could use pattern matching: `if (obj instanceof Foo foo)`
- Sealed interface opportunities for closed type hierarchies
- Text blocks for multiline strings (SQL, JSON, templates)
- `Optional` misuse: calling `.get()` without `.isPresent()` check, or using `Optional` as a method parameter or field type
- `Stream` operations that could be simplified (`.stream().collect(Collectors.toList())` → `.stream().toList()`)

**Null Safety:**
- Methods returning `null` where `Optional` would be clearer
- Dereferencing a value that could be null without a null check
- `@Nullable` parameters passed to methods that don't handle null
- Primitive wrapper types (`Integer`, `Long`) where primitives suffice (avoids NPE risk + boxing overhead)

**Immutability & Thread Safety:**
- Mutable fields in classes used across threads without synchronization
- Returning mutable internal collections directly — use `Collections.unmodifiableList()` or `List.copyOf()`
- `Date` / `Calendar` usage — use `java.time` types (`Instant`, `LocalDateTime`, `ZonedDateTime`)
- `StringBuilder` or mutable state in places where immutable alternatives exist

**Logging:**
- String concatenation in log statements (`log.debug("User " + id)`) — use parameterized logging (`log.debug("User {}", id)`)
- Logging sensitive data (passwords, tokens, PII) at any level
- Missing `log.isDebugEnabled()` guard before expensive debug log construction
- Catching and logging the same exception at multiple layers

**Resource Management:**
- Streams, connections, readers/writers not in try-with-resources blocks
- `EntityManager` or `Session` opened manually without close guarantee
- Connection pools without proper max-size or timeout configuration

**Spring-Specific Code Smells:**
- `@Component` / `@Service` / `@Repository` on classes that should be plain POJOs
- Multiple `@Configuration` classes in the same package doing unrelated things
- `@Value` injection of complex expressions — extract to `@ConfigurationProperties` class
- Hardcoded `@Scheduled` cron expressions instead of externalized config
- `@Autowired` on more than one constructor (ambiguous injection)

---

## Suppressions — DO NOT flag these

- Threshold/constant value choices (these are tuned empirically)
- "Add Javadoc" suggestions
- Style-only changes with no functional impact (formatting, import order)
- Test utility patterns that look unusual but work correctly
- Lombok-generated code smells (Lombok handles it)
- Framework-required patterns that look odd but are necessary (e.g., empty no-arg constructors for JPA)
- ANYTHING already addressed in the diff
