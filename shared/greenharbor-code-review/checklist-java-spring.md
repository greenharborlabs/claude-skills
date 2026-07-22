# Java / Spring Boot Pre-Landing Review Checklist

## Instructions

Apply these Java/Spring checks to the active review input and use the output format from `SKILL.md`.
Only flag real problems with specific `file:line` evidence. Skip anything that is fine or already addressed.

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
- Declared request constraints that never execute and have no equivalent boundary validation
- Required or bounded input accepted without annotation-based or equivalent programmatic validation

#### Broken Object-Level Authorization (BOLA)
- User-supplied IDs used to fetch records without ownership/authorization check
- Missing `@PreAuthorize` or manual authorization on endpoints modifying user-specific data

#### Missing Authorization
- Sensitive endpoints without effective filter-chain, annotation, or programmatic authorization
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
- Missing failure handling where operation semantics and observed dependency risk require it; do not demand retries or circuit breakers universally

#### Pagination
- Repository methods returning `List<Entity>` for unbounded result sets — use `Page` or `Slice`
- Missing default/max page size limits

#### Virtual Thread Compatibility
- On JDK 23 or earlier, blocking inside `synchronized` may pin virtual threads; JDK 24+ eliminated ordinary monitor pinning
- Remaining native-frame pinning, unsafe thread-local assumptions, or unbounded virtual-thread work when virtual threads are actually enabled

#### Error Handling
- Unhandled or inconsistent observable error contracts; `@ControllerAdvice` is one option, not a requirement
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
- Methods, branches, or nesting whose demonstrated cognitive load or mixed responsibilities obscure behavior
- Classes with multiple unrelated responsibilities or changes that materially deepen coupling
- Non-exhaustive `switch` logic that misses a reachable case; do not require `default` for compiler-checked exhaustive switches

**Modern Java (only when supported and materially clearer):**
- Mutable value/transport types that would become safer and simpler as records
- Type branches or closed hierarchies where pattern matching or sealed types reduce real complexity
- Text blocks that improve genuinely multiline strings
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
