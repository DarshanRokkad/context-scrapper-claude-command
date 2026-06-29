# /explore-service — Spring Boot Service DNA Extractor

You are executing the `/explore-service` command. Your goal is to analyze this Spring Boot
microservice repository and generate a complete `.claude/` directory so that every future
Claude Code session in this repo generates service-native code automatically.

Run this command once per repository. Commit the `.claude/` folder to version control.

---

## Phase 1: Orchestrator — Repository Surface Scan

Before spawning agents, read the following to build a shared context snapshot:

1. Read `pom.xml` or `build.gradle.kts` or `build.gradle` (whichever exists)
   - Extract: `artifactId` (service name), `description`, key dependencies
     (Spring Web, Spring Data JPA, Kafka, Lombok, MapStruct, Flyway, etc.)

2. List all directories under `src/main/java` recursively to see the full package tree.
   Do not read file contents yet — just get the structural map.

3. Read `README.md` if it exists. Extract the service's purpose in 1–2 sentences.

Store these findings. You will pass them to each sub-agent as starting context.

**Scan exclusions — always skip these paths in every phase:**
`target/`, `build/`, `.git/`, `.idea/`, `generated-sources/`, `node_modules/`,
any file ending in `.class`, `.jar`, `.war`

**Sampling limits — enforce in every agent:**
- Read at most **5 files per architectural layer** (e.g., 5 controllers, 5 services)
- Read at most **20 files per extraction concern**
- When multiple files qualify, prefer files with more methods and richer annotation sets

---

## Phase 2: Spawn 7 Sub-Agents in Parallel

Use the Agent tool to spawn all seven agents simultaneously. Pass each agent the shared
context from Phase 1 (service name, dependencies, package tree, purpose).

**Every agent must follow these output rules:**

- **Frequency reporting:** For each pattern detected, state what percentage of scanned
  files use it. Example: "`@Transactional` at class-level: 80% (8/10 service files),
  method-level: 20% (2/10)". The majority pattern is the preferred pattern.
- **Confidence levels:** Tag every extracted section as:
  - `[High]` — found in 5+ locations
  - `[Medium]` — found in 2–4 locations
  - `[Low]` — found in 1 location or inferred from structure
- Apply sampling limits and scan exclusions defined in Phase 1.

---

### Sub-Agent 1: Architecture Agent

**Your job:** Extract the structural and transactional DNA of this service.

**Step 1 — Package Layer Scan**
- Map all packages under `src/main/java`
- Identify the layer naming convention this service uses:
  - Is it `controller` or `adapter` or `web`?
  - Is it `service` or `usecase` or `application`?
  - Is it `repository` or `port` or `persistence`?
  - Is it `domain` or `entity` or `model`?
- Determine module organization: **by layer** (all controllers in one package) vs
  **by feature** (each feature has its own controller + service + repo sub-packages)

**Step 2 — Layer Boundary Rules**
- Read 2–3 representative files from each identified layer
- Extract: which Spring annotations are used at each layer
  (`@RestController`, `@Service`, `@Repository`, `@Component`, `@Bean`)
- Note: does the service layer depend on the repository directly or through interfaces?

**Step 3 — Transaction Strategy Scan**
- Search for all `@Transactional` usages across `src/main/java`
- Determine: is `@Transactional` applied at the **class level** or **method level**
  in the Service layer?
- Check: do read-only methods explicitly declare `@Transactional(readOnly = true)`?
- Extract one real code example showing the transaction pattern used

**Write your findings to `.claude/architecture.md` using this structure:**

```markdown
# Architecture

## Service Overview
[service name and 1-sentence purpose]

## Package Layer Map
[table: layer name → package path → Spring annotation used]

## Module Organization
[by layer or by feature — explain with example]

## Layer Dependency Direction
[how layers depend on each other]

## Transaction Strategy
[class-level vs method-level @Transactional, readOnly usage]

### Example
[real code snippet from this repo]
```

---

### Sub-Agent 2: Conventions Agent

**Your job:** Capture the exact coding syntax and style patterns used in this service.

**Step 1 — Logging Scan**
- Find all Logger/log field declarations (SLF4J, Log4j2, Lombok `@Slf4j`)
- Find at least 5 log call sites at different levels (info, debug, warn, error)
- Extract the exact syntax used, especially how context is passed:
  - Simple string: `log.info("Processing payment")`
  - Placeholder: `log.info("Processing payment for id={}", id)`
  - Structured key-value: `log.info("Processing payment", kv("paymentId", id))`
  - MDC usage if present

**Step 2 — Validation Scan**
- Find usages of `@Valid`, `@Validated`, `@NotNull`, `@NotBlank`, `@Size`, `@Pattern`,
  and any custom `@Constraint` validators
- Determine: where is validation enforced — controller method parameters, request DTOs,
  service layer, or all of the above?
- Note: does the service use method-level validation via `@Validated` on service classes?

**Step 3 — DTO and Mapper Scan**
- Find request and response DTO/payload classes
- Extract the naming convention:
  `XxxRequest` / `XxxResponse`? `XxxDto`? `XxxPayload`? `XxxCommand` / `XxxView`?
- Identify the mapping strategy used:
  - MapStruct `@Mapper` interfaces?
  - Manual `XxxMapper` classes with static or instance methods?
  - Inline mapping inside service methods?
- Read 2–3 real DTO classes and their mappers as examples

**Step 4 — Naming Convention Scan**
- Extract domain-specific terminology from class and method names
  (e.g., does the service say `customerId` or `userId`? `transaction` or `payment`?)
- Note HTTP method naming patterns on controllers
- Note repository method naming patterns

**Step 5 — Configuration Property Style Scan**
- Search for `@Value("${...}")` usages across `src/main/java`
- Search for classes annotated with `@ConfigurationProperties`
- Determine: does this service prefer **direct `@Value` injection** or
  **type-safe `@ConfigurationProperties` beans**?
- If `@ConfigurationProperties` is used, note the prefix pattern and whether the class
  uses records, `@ConstructorBinding`, or plain setters
- Extract one real example of the preferred configuration style

**Write your findings to `.claude/conventions.md` using this structure:**

```markdown
# Conventions

## Logging
[logger declaration style]
[exact syntax with examples at info / debug / warn / error level]

## Validation
[where validation is enforced]
[annotations used with examples]

## DTO Pattern
[naming convention]
[mapping strategy with example]

## Naming Conventions
[domain terminology]
[key naming patterns]

## Configuration Property Style
[preferred approach: @Value or @ConfigurationProperties]

### Example
[real code snippet from this repo]
```

---

### Sub-Agent 3: Error Handling Agent

**Your job:** Map the complete exception and error response architecture.

**Step 1 — Global Exception Handler**
- Find the `@RestControllerAdvice` or `@ControllerAdvice` class
- Read it entirely
- Extract: which exception types are handled, what HTTP status each maps to,
  what response body is returned

**Step 2 — Exception Hierarchy**
- Find all custom exception classes under `src/main/java`
- Build the hierarchy: what is the base exception? what domain-specific subclasses exist?
- Note: do exceptions carry an error code field? Is the error code an enum?
- Note: are exceptions checked or unchecked (extend RuntimeException or Exception)?

**Step 3 — Error Response Model**
- Find the error response payload class (the object serialized in 4xx/5xx bodies)
- Extract its fields: `code`? `message`? `timestamp`? `details`? `traceId`?
- Note: is there a generic response envelope wrapping both success and error payloads?

**Write your findings to `.claude/error-handling.md` using this structure:**

```markdown
# Error Handling

## Global Exception Handler
[class name and package]
[table: exception type → HTTP status → response format]

## Exception Hierarchy
[base exception class]
[domain exception subclasses]
[error code enum if present]

## Error Response Model
[class name]
[fields and their types]

### Example — throwing a domain exception
[real code snippet from this repo]

### Example — error response payload
[real JSON or class snippet]
```

---

### Sub-Agent 4: Integration & Utilities Agent

**Your job:** Capture how this service talks to the outside world and what shared utilities it provides.

**Step 1 — Outbound HTTP Client Scan**
- Search for `@FeignClient` interfaces — read all of them fully
  - Extract: how the interface is declared, how the base URL is configured
    (hardcoded, `@Value`, `@ConfigurationProperties`, service discovery?)
  - Extract: how request/response types are defined (shared DTOs? client-specific models?)
  - Extract: how errors from downstream calls are handled
    (`ErrorDecoder`? `fallback`? try-catch in the service layer?)
- If no Feign, search for `RestTemplate` bean declarations and injection patterns
- If no RestTemplate, search for `WebClient` builder configuration and usage
- Note which HTTP client pattern this service standardizes on

**Step 2 — Utility Classes Scan**
- Find classes in packages named `util`, `utils`, `helper`, `common`, `shared`
- For each utility class:
  - Note whether it is a static utility class or a Spring bean
  - Summarize its purpose in one line (e.g., `DateUtils` — static methods for epoch
    conversion, `PaginationHelper` — builds `Pageable` from request params)
- Find any custom annotations defined in the codebase (not Spring built-ins)
  — extract their purpose and where they are applied
- Find any constants classes or enums used across multiple packages

**Write your findings to `.claude/integrations.md` using this structure:**

```markdown
# Integrations & Utilities

## Outbound HTTP Clients
[which client library is used: Feign / RestTemplate / WebClient]
[how base URLs are configured]
[how downstream errors are handled]

### Example — Feign client declaration (or RestTemplate/WebClient equivalent)
[real code snippet from this repo]

## Utility Classes
[table: class name → type (static/bean) → purpose]

## Custom Annotations
[table: annotation → where applied → purpose]

## Constants & Enums
[shared constants/enum classes and what they represent]
```

---

### Sub-Agent 5: Persistence Agent

**Your job:** Extract how entities are modeled and persisted in this service.

**Step 1 — Entity Structure Scan**
- Find all `@Entity` classes (max 5)
- Identify: is there a base entity class (e.g., `BaseEntity`, `AuditEntity`) that others extend?
- Extract base entity fields: `id`, `createdAt`, `updatedAt`, `version`, any other shared fields
- Extract ID generation strategy: `@GeneratedValue(strategy = ...)` — IDENTITY, SEQUENCE, UUID?

**Step 2 — Audit & Soft Delete Scan**
- Check for JPA auditing: `@CreatedDate`, `@LastModifiedDate`, `@CreatedBy`, `@LastModifiedBy`
  — is `@EnableJpaAuditing` present in a config class?
- Check for soft delete: is there a `deletedAt` or `isDeleted` field?
  Look for `@SQLDelete` or `@Where` annotations indicating soft delete pattern

**Step 3 — Relationship Scan**
- Find `@OneToMany`, `@ManyToOne`, `@ManyToMany`, `@OneToOne` usages
- For each: extract fetch type (LAZY vs EAGER), cascade type, join column naming

**Write your findings to `.claude/persistence.md` using this structure:**

```markdown
# Persistence

## Base Entity [confidence]
[base class name, shared fields, inheritance strategy]

## ID Generation [confidence]
[strategy with example — frequency across entity classes]

## Audit Fields [confidence]
[which audit annotations are used, config class that enables them]

## Soft Delete [confidence]
[pattern used — deletedAt field / @SQLDelete / @Where / none]

## Relationship Conventions [confidence]
[default fetch type, cascade rules, join column naming pattern]
```

---

### Sub-Agent 6: Testing Agent

**Your job:** Extract test conventions so generated tests are indistinguishable from
hand-written ones.

**Step 1 — Test Framework Scan**
- Confirm JUnit version (4 vs 5) from `pom.xml` / `build.gradle`
- Find which Spring test slices are used: `@SpringBootTest`, `@WebMvcTest`,
  `@DataJpaTest`, `@ExtendWith(MockitoExtension.class)`
- Identify Mockito style: `@Mock` + `@InjectMocks`, or `@MockBean`, or BDDMockito (`given/willReturn`)

**Step 2 — Test Naming & Structure Scan**
- Find 3–5 test classes under `src/test/java` (max 5)
- Extract: test class naming convention (`XxxTest`? `XxxSpec`? `XxxShould`?)
- Extract: test method naming convention (`methodName_scenario_expectedResult`?
  `given_when_then`? `should_do_something_when_condition`?)
- Note: is `@DisplayName` used for readable test names?

**Step 3 — Fixture & Assertion Scan**
- Find how test data is set up: `@BeforeEach` builders, static factory methods,
  separate `XxxFixture` / `XxxMother` classes?
- Identify assertion library: AssertJ (`assertThat`), Hamcrest (`assertThat`/`is`),
  or plain JUnit assertions?

**Write your findings to `.claude/testing.md` using this structure:**

```markdown
# Testing

## Framework & Annotations [confidence]
[JUnit version, test slice annotations used, Mockito style]

## Test Naming Convention [confidence]
[class naming pattern, method naming pattern — with frequency]

## Fixture Strategy [confidence]
[how test data is created — @BeforeEach / builders / fixture classes]

## Assertion Style [confidence]
[AssertJ / Hamcrest / JUnit — with example]
```

---

### Sub-Agent 7: Security Agent

**Your job:** Extract how this service secures its endpoints and accesses user context.
*(If no security configuration is found, write `.claude/security.md` with a single line:
"No Spring Security configuration found in this service.")*

**Step 1 — Security Config Scan**
- Find `SecurityFilterChain` bean — read it entirely
- Extract: which URL patterns are permitted without auth, which require authentication
- Note: is JWT used? Find JWT filter class or `JwtDecoder` bean

**Step 2 — Method Security Scan**
- Search for `@PreAuthorize`, `@PostAuthorize`, `@Secured` usages
- Extract: what role/permission strings are used, naming convention for authorities

**Step 3 — Principal Access Scan**
- Find how the current user is accessed: `SecurityContextHolder.getContext().getAuthentication()`?
  A custom `@CurrentUser` annotation? `Principal` method parameter?
- Extract the exact pattern used with a real example

**Write your findings to `.claude/security.md` using this structure:**

```markdown
# Security

## Endpoint Protection [confidence]
[which paths are open vs authenticated — from SecurityFilterChain]

## JWT / Token Strategy [confidence]
[how tokens are validated, what claims are extracted]

## Method-Level Security [confidence]
[@PreAuthorize patterns used, permission naming convention]

## Accessing Current User [confidence]
[exact pattern — SecurityContextHolder / @CurrentUser / Principal param]

### Example
[real code snippet showing how user context is accessed]
```

---

## Phase 3: Skills Generator Agent

After all 7 parallel agents complete, spawn one final agent with these instructions:

---

### Part A — Extract Real Examples First

Before writing any skill file, find the single best real example of each component type
in this repo and copy the relevant excerpt to `.claude/examples/`. These files are the
ground truth that skill files will reference — they must come from actual production code
in this repo, not invented.

For each component below, score candidate files and pick the highest-scoring one.
Copy the full class exactly as-is — do not modify or summarize.

**Scoring criteria (apply to every example selection):**
- +3 — has 3 or more methods
- +2 — uses domain-specific exception types (not generic RuntimeException)
- +2 — uses the service's logging pattern (not System.out)
- +2 — has the full annotation set for its layer
- +1 — uses constructor injection (not @Autowired fields)
- −2 — is a test class or fixture
- −2 — is in a `generated-sources` or `generated` package

Pick the candidate with the highest score. If scores are tied, prefer the file
with more lines.

| Extract | Write to |
|---------|----------|
| One representative REST controller class | `.claude/examples/controller.java` |
| One representative Service class | `.claude/examples/service.java` |
| One representative Repository interface | `.claude/examples/repository.java` |
| One request DTO + its response DTO + mapper | `.claude/examples/dto.java` |
| One `@Entity` class (prefer one that extends a base entity) | `.claude/examples/entity.java` |
| One test class (prefer a service unit test) | `.claude/examples/test.java` |
| One `@FeignClient` interface *(skip if not present)* | `.claude/examples/feign-client.java` |
| The `@RestControllerAdvice` class | `.claude/examples/exception-handler.java` |

Do not modify the code. Copy it exactly from the source file.

---

### Part B — Write Skill Files

Now write the skill files. Each skill must be **under 150 lines**. Do not embed long
code snippets — reference the extracted example files instead. Follow this structure
for every skill:

```
## Job
One sentence: what failure mode this skill prevents.

## Before You Start
Read `.claude/avoid.md` — do not generate any pattern listed there.
Read `.claude/examples/[file].java` — this is the real pattern used in this repo.

## Rules
DO: [active, specific commands — 8 to 12 bullets]
NEVER: [explicit bans — 4 to 6 bullets]

## Steps
Numbered steps to follow when generating this component.
Keep each step to one sentence.
```

Every skill file must open with the "Before You Start" section referencing both
`avoid.md` and the relevant example file. No exceptions.

---

### `.claude/skills/generate-controller.md`

**Job:** Prevent Claude from generating controllers that use wrong annotations,
invent exception handling, or log with a style not used in this repo.

**Reference Example:** `.claude/examples/controller.java`

**Rules — fill these in from the extracted artifacts:**

DO:
- Place the controller in the exact package path shown in `.claude/architecture.md`
- Use the class-level annotations seen in the reference example (copy them exactly)
- Use the logging syntax from `.claude/conventions.md` — no other style
- Delegate all business logic to the service layer
- Return the response type pattern shown in the reference example
- Let exceptions propagate — the global handler in `.claude/error-handling.md` catches them

NEVER:
- Catch exceptions inside a controller method — no try/catch blocks here
- Use `System.out.println` or any logger style not in `.claude/conventions.md`
- Add `@Autowired` field injection — use constructor injection only
- Invent a new response wrapper class not already present in this codebase

**Steps:**
1. Read `.claude/examples/controller.java`
2. Read `.claude/conventions.md` for exact logging syntax
3. Copy the class structure and annotation set from the example
4. Replace domain-specific names with the new feature's names
5. Delegate to the service layer for all logic

---

### `.claude/skills/generate-service.md`

**Job:** Prevent Claude from placing `@Transactional` incorrectly, using wrong
logging patterns, or throwing exceptions that don't belong to this service's hierarchy.

**Reference Example:** `.claude/examples/service.java`

**Rules — fill these in from the extracted artifacts:**

DO:
- Place `@Transactional` at class-level or method-level exactly as described in `.claude/architecture.md`
- Mark read-only methods with `@Transactional(readOnly = true)` if this repo does so
- Use constructor injection only — `@RequiredArgsConstructor` or explicit constructor
- Log method entry and exit using the syntax in `.claude/conventions.md`
- Throw domain exceptions from the hierarchy in `.claude/error-handling.md`

NEVER:
- Place `@Transactional` on repository or controller classes
- Catch and swallow exceptions silently
- Use `new RuntimeException()` — always use the domain exception base class
- Add `@Autowired` field injection

**Steps:**
1. Read `.claude/examples/service.java`
2. Read `.claude/architecture.md` for transaction strategy
3. Read `.claude/error-handling.md` for the exception hierarchy
4. Copy the class structure from the example
5. Apply the correct `@Transactional` placement per this repo's strategy

---

### `.claude/skills/generate-repository.md`

**Job:** Prevent Claude from extending the wrong base interface or writing queries
in a style inconsistent with this repo.

**Reference Example:** `.claude/examples/repository.java`

**Rules — fill these in from the extracted artifacts:**

DO:
- Extend the same base interface shown in the reference example
- Follow the method naming convention seen in existing repositories
- Use `@Query` only when a derived method name would be unreadably long
- Use `Pageable` for list operations if the repo does so

NEVER:
- Mix JPQL and native queries without a clear reason
- Write queries that can be expressed as derived method names
- Annotate the repository with `@Service` or `@Component`

**Steps:**
1. Read `.claude/examples/repository.java`
2. Extend the same interface it extends
3. Name new methods following the same verb pattern (findBy, existsBy, deleteBy)
4. Add `@Query` only if needed

---

### `.claude/skills/generate-dto.md`

**Job:** Prevent Claude from inventing DTO names, field annotation styles,
or mapper patterns that differ from what this repo uses.

**Reference Example:** `.claude/examples/dto.java`

**Rules — fill these in from the extracted artifacts:**

DO:
- Use the naming convention from `.claude/conventions.md` (e.g., `XxxRequest` / `XxxResponse`)
- Use the same field annotation style as the reference example (Lombok, records, etc.)
- Place validation annotations on fields exactly as shown in the reference
- Use the same mapping approach (MapStruct interface, manual mapper class, or inline)

NEVER:
- Invent a new naming pattern for DTOs not already in use
- Add Jackson annotations (`@JsonProperty`) if the repo does not use them
- Put business logic inside a DTO class

**Steps:**
1. Read `.claude/examples/dto.java`
2. Name the new DTO following the same convention
3. Copy the field annotation style
4. Write the mapper in the same location and style as the example

---

### `.claude/skills/generate-feign-client.md`
*(Only write this file if `.claude/examples/feign-client.java` exists)*

**Job:** Prevent Claude from declaring Feign clients with wrong configuration,
inventing fallback patterns not in this repo, or placing client interfaces
in the wrong package.

**Reference Example:** `.claude/examples/feign-client.java`

**Rules — fill these in from the extracted artifacts:**

DO:
- Place the client interface in the same package as existing clients
- Configure the base URL using the same approach as the reference (property key, config class)
- Define request/response types using the same model location pattern
- Handle downstream errors using the same strategy shown in `.claude/integrations.md`

NEVER:
- Add a `fallback` or `fallbackFactory` unless the reference example uses one
- Hardcode URLs — always read from configuration
- Mix this service's domain DTOs with client-specific response models without a mapper

**Steps:**
1. Read `.claude/avoid.md`
2. Read `.claude/examples/feign-client.java`
3. Read `.claude/integrations.md` for error-handling strategy
4. Declare the interface following the reference exactly
5. Wire the configuration property using the same key pattern

---

### `.claude/skills/generate-entity.md`

**Job:** Prevent Claude from inventing base entity inheritance, ID generation strategies,
audit annotations, or relationship fetch types that differ from this repo's conventions.

**Before You Start:**
Read `.claude/avoid.md`.
Read `.claude/examples/entity.java` — this is the real entity pattern used in this repo.

**Rules — fill these in from `.claude/persistence.md`:**

DO:
- Extend the same base entity class used across this repo (if one exists)
- Use the ID generation strategy shown in `.claude/persistence.md`
- Include audit fields only if the base entity or `@EnableJpaAuditing` pattern covers them
- Use LAZY fetch type for all `@OneToMany` and `@ManyToMany` by default
- Follow the join column naming convention shown in existing entities
- Apply soft delete pattern (`deletedAt` / `@SQLDelete`) if this repo uses it

NEVER:
- Use EAGER fetch on collection relationships
- Redefine audit fields (`createdAt`, `updatedAt`) that are already in the base entity
- Use `@Table` without following the existing table naming convention
- Generate an entity without inheriting from the base entity if one exists in this repo

**Steps:**
1. Read `.claude/avoid.md`
2. Read `.claude/examples/entity.java`
3. Read `.claude/persistence.md` for base entity, ID strategy, and audit setup
4. Extend the base entity class
5. Add only domain-specific fields — do not duplicate inherited fields
6. Apply relationship annotations following the fetch/cascade convention

---

### `.claude/skills/generate-test.md`

**Job:** Prevent Claude from writing tests in the wrong JUnit version, wrong mock style,
wrong naming convention, or wrong assertion library for this repo.

**Before You Start:**
Read `.claude/avoid.md`.
Read `.claude/examples/test.java` — this is the real test pattern used in this repo.

**Rules — fill these in from `.claude/testing.md`:**

DO:
- Use the test slice annotation appropriate for the component being tested
  (`@WebMvcTest` for controllers, `@ExtendWith(MockitoExtension.class)` for services)
- Follow the method naming convention shown in `.claude/testing.md`
- Use the assertion library this repo uses (AssertJ / Hamcrest / JUnit)
- Create test fixtures using the same strategy (builders / `@BeforeEach` / fixture class)
- Mock dependencies using the same style (`@MockBean` vs `@Mock` + `@InjectMocks`)

NEVER:
- Mix JUnit 4 and JUnit 5 annotations
- Use `@SpringBootTest` for unit tests — only use it for integration tests if this repo does
- Use `System.out.println` for debugging in tests
- Write tests without assertions

**Steps:**
1. Read `.claude/avoid.md`
2. Read `.claude/examples/test.java`
3. Read `.claude/testing.md` for naming convention, mock style, and assertion library
4. Choose the correct test slice for the component under test
5. Set up fixtures following the repo's strategy
6. Name the test class and methods following the detected convention

---

### Part C — Generate `.claude/avoid.md`

Read all context artifacts produced so far:
`.claude/architecture.md`, `.claude/conventions.md`, `.claude/error-handling.md`,
`.claude/integrations.md`, `.claude/persistence.md`, `.claude/testing.md`, `.claude/security.md`

Then write `.claude/avoid.md` capturing patterns that exist in the codebase but
should not be replicated:

```markdown
# Patterns to Avoid

## Deprecated Code Patterns
[patterns found in old files but replaced — e.g., "RestTemplate exists in legacy classes
but has been replaced by Feign — do not use RestTemplate in new code"]

## Wrong Injection Styles
[e.g., "@Autowired field injection exists in 3 old classes — always use constructor injection"]

## Banned Exception Types
[e.g., "do not throw RuntimeException directly — use the domain exception hierarchy"]

## Anti-Patterns Observed
[any other patterns found in the codebase that are clearly wrong or inconsistent
with the majority convention — inferred from low-frequency / low-confidence findings]
```

Every skill file references this before generating. If a pattern appears in `avoid.md`,
it must not appear in any generated code regardless of what the user asks.

---

## Phase 4: Write CLAUDE.md and Metadata

**First, write `.claude/metadata.json`:**

```json
{
  "serviceName": "[artifactId from pom.xml]",
  "generatedAt": "[ISO 8601 timestamp — current date/time]",
  "commitSha": "[output of: git rev-parse --short HEAD, or 'unknown' if git not available]",
  "scanVersion": "1.0"
}
```

**Then write `.claude/CLAUDE.md`.** Keep it concise — it loads every session.

Use this exact structure (fill in the bracketed sections from what was extracted):

```markdown
# [Service Name] — Claude Code Context

## What This Service Does
[1–2 sentences from README or pom description]

## Key Dependencies
[bullet list of the most relevant ones: Spring Web, JPA, etc.]

## Build & Run Commands
- Build: `mvn clean install` / `./gradlew build`
- Test: `mvn test` / `./gradlew test`
- Run: `mvn spring-boot:run` / `./gradlew bootRun`

## Architecture Summary
[2–3 sentences: layer structure, module organization, transaction strategy]

→ Full details: read `.claude/architecture.md`

## Key Conventions
[2 sentences: logging style, DTO naming, config injection style]

→ Full details: read `.claude/conventions.md`

## Error Handling Summary
[1–2 sentences: base exception class, where the global handler lives]

→ Full details: read `.claude/error-handling.md`

## Integrations Summary
[1–2 sentences: which HTTP client is used, key util classes]

→ Full details: read `.claude/integrations.md`

## Persistence Summary
[1–2 sentences: base entity class, ID strategy, soft delete if used]

→ Full details: read `.claude/persistence.md`

## Testing Summary
[1–2 sentences: JUnit version, mock style, assertion library]

→ Full details: read `.claude/testing.md`

## Security Summary
[1 sentence: how endpoints are protected, how current user is accessed]

→ Full details: read `.claude/security.md`

## Patterns to Avoid
Before generating any code, read `.claude/avoid.md`.
It lists deprecated patterns and anti-patterns found in this repo — do not replicate them.

## Generating New Components

Before generating any new component, read the relevant skill and its reference example:

| Task | Skill | Reference |
|------|-------|-----------|
| New REST endpoint or controller | `.claude/skills/generate-controller.md` | `.claude/examples/controller.java` |
| New service class | `.claude/skills/generate-service.md` | `.claude/examples/service.java` |
| New repository or query | `.claude/skills/generate-repository.md` | `.claude/examples/repository.java` |
| New DTO or mapper | `.claude/skills/generate-dto.md` | `.claude/examples/dto.java` |
| New entity | `.claude/skills/generate-entity.md` | `.claude/examples/entity.java` |
| New test | `.claude/skills/generate-test.md` | `.claude/examples/test.java` |
| New outbound HTTP client | `.claude/skills/generate-feign-client.md` | `.claude/examples/feign-client.java` |

NEVER introduce patterns, annotations, logging styles, or exception types
that are not already present in this codebase.
```

---

## Done

Report a summary of what was extracted, listing:
- Each `.claude/` file created
- Which optional skill files were written vs skipped (feign-client, etc.)
- Top 3–5 most distinctive patterns found, each with frequency and confidence
  (e.g., "method-level @Transactional: 80% [High]", "@ConfigurationProperties: 100% [High]",
  "soft delete via deletedAt field: [Medium]")
- Any patterns added to `avoid.md` and why
