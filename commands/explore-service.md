# /explore-service — Spring Boot Service DNA Extractor

You are executing the `/explore-service` command. Your goal is to analyze this Java Spring Boot
microservice repository (Maven or Gradle, single or multi-module) and
generate a complete `.claude/` directory so that every future Claude Code session in this
repo generates service-native code automatically.

Run this command once per repository. Commit the `.claude/` folder to version control.

---

## Shared Rules — Apply in Every Phase and Every Agent

These rules are defined once here. Every sub-agent must follow them without being told again.

### Scan Exclusions
Always skip: `target/`, `build/`, `.git/`, `.idea/`, `generated-sources/`, `generated/`,
`node_modules/`, and any file ending in `.class`, `.jar`, `.war`.

### Sampling Limits
- Read at most **5 files per architectural layer** (e.g., 5 controllers, 5 services)
- Read at most **20 files per extraction concern**
- When multiple files qualify, prefer files with more methods and richer annotation sets

### Frequency Reporting
For each pattern detected, state what percentage of scanned files use it.
Example: "`@Transactional` at class-level: 80% (8/10 service files), method-level: 20% (2/10)."
The majority pattern is the preferred pattern.

### Confidence Levels
Tag every extracted section as:
- `[High]` — found in 5+ locations
- `[Medium]` — found in 2–4 locations
- `[Low]` — found in 1 location or inferred from structure

---

## Phase 0: Repo Profile Detection (Orchestrator)

Before reading source code, detect the repository's fundamental characteristics.
The output of this phase — the REPO PROFILE — is passed verbatim to every sub-agent.
No agent should make assumptions about build tool or source paths.

### Step 1 — Build Tool Detection
Check for these files at the repository root:
- `pom.xml` → build tool is **Maven**
  - Also check for `mvnw` → wrapper command is `./mvnw`, else `mvn`
- `build.gradle.kts` → build tool is **Gradle**
  - Also check for `gradlew` → wrapper command is `./gradlew`, else `gradle`
- `build.gradle` → build tool is **Gradle**
  - Same wrapper check

Read whichever build file is found. Extract:
- Service/project name (`<artifactId>` for Maven; `rootProject.name` from `settings.gradle*` for Gradle)
- Spring Boot version
- All declared dependencies (used in Step 4)

### Step 2 — Module Structure Detection
`source_roots` always points to Java source paths. Set the initial root to `src/main/java`.

**Maven:** If root `pom.xml` contains a `<modules>` section → **multi-module**.
List each `<module>` child. Replace `source_roots` with `{module}/src/main/java` for each module.

**Gradle:** If `settings.gradle` or `settings.gradle.kts` contains `include(...)` calls
→ **multi-module**. List each included project. Replace `source_roots` with their
`{project}/src/main/java` paths.

If no module section found → **single-module**. `source_roots` = `src/main/java`.

### Step 3 — Framework Detection
From the build file dependencies, detect presence of:
Kafka, Spring Security, Feign/OpenFeign, Flyway, Liquibase, Testcontainers,
MapStruct, Lombok, Spring Data JPA, WebClient/WebFlux.

### Step 4 — Service Purpose
Read `README.md` if it exists. Extract the service purpose in 1–2 sentences.
If no README: use `<description>` from `pom.xml` or description comment in `build.gradle*`.

### Produce the REPO PROFILE
Store this block and pass it verbatim to every sub-agent prompt:

```
REPO PROFILE
service:          [name]
build_tool:       maven | gradle
wrapper_cmd:      ./mvnw | ./gradlew | mvn | gradle
module_structure: single | multi
modules:          [comma-separated module directory names, or "n/a"]
source_roots:     [comma-separated list of all detected source paths]
spring_boot:      [version]
frameworks:       [comma-separated list of detected frameworks]
purpose:          [1–2 sentence service description]
```

---

## Phase 1: Package Tree Scan (Orchestrator)

Using the `source_roots` from the REPO PROFILE, list all directories recursively under
each source root — do not read file contents yet. Identify the top-level package common
to all files (e.g., `com.example.payments`).

Append to the REPO PROFILE:

```
package_root:     [top-level package, e.g., com.example.payments]
package_tree:     [indented tree — 2 levels deep per source root]
```

---

## Phase 2: Spawn 7 Sub-Agents in Parallel

Use the Agent tool to spawn all seven agents simultaneously.

> **IMPORTANT:** Copy the complete REPO PROFILE block (including `package_root` and
> `package_tree`) into each agent's prompt verbatim. This is the only shared context
> agents receive — they must derive all scan paths from `source_roots` and all
> conditional behavior from `frameworks`.

All agents must apply the Shared Rules from the top of this command.

---

### Sub-Agent 1: Architecture Agent

**Context required:** Full REPO PROFILE block.

**Your job:** Extract the structural and transactional DNA of this service.

**Step 1 — Package Layer Scan**
For each path in `source_roots`, map all packages and identify the layer naming convention:
- Web layer: `controller`? `adapter`? `web`? `resource`?
- Business layer: `service`? `usecase`? `application`?
- Data layer: `repository`? `port`? `persistence`? `infrastructure`?
- Model layer: `domain`? `entity`? `model`?

Determine module organization:
- **By layer** — all controllers in one package, all services in another
- **By feature** — each feature has its own controller + service + repo sub-packages

If multi-module: note which module owns which layer
(e.g., `api/` has controllers, `service/` has business logic, `persistence/` has repositories).

**Step 2 — Layer Boundary Rules**
Read 2–3 representative files from each identified layer.
Extract: which Spring annotations appear at each layer.
Note: does the service layer depend on repositories directly or through interfaces?

**Step 3 — Transaction Strategy Scan**
Search for all `@Transactional` usages across all `source_roots`.
- Determine: class-level or method-level in the service layer?
- Check: do read-only methods use `@Transactional(readOnly = true)`?
- Extract one real code example showing the transaction pattern used.

**Write your findings to `.claude/architecture.md`:**

```markdown
# Architecture

## Service Overview
[service name and 1-sentence purpose]

## Package Layer Map
[table: layer name → package path → Spring annotation used]

## Module Organization
[by layer or by feature — explain with one example;
for multi-module: which module owns which layer]

## Layer Dependency Direction
[how layers depend on each other]

## Transaction Strategy [confidence]
[class-level vs method-level @Transactional, readOnly usage]

### Example
[real code snippet from this repo]
```

---

### Sub-Agent 2: Conventions Agent

**Context required:** Full REPO PROFILE block.

**Your job:** Capture the exact coding syntax and style patterns used in this service.

**Step 1 — Logging Scan**
Find all logger field declarations across all `source_roots`:
`LoggerFactory.getLogger(...)` or Lombok `@Slf4j`.

Find at least 5 log call sites at different levels. Extract the exact syntax:
- Simple string: `log.info("Processing payment")`
- Placeholder: `log.info("Processing payment for id={}", id)`
- Structured key-value: `log.info("Processing payment", kv("paymentId", id))`
- MDC usage if present

**Step 2 — Validation Scan**
Find usages of `@Valid`, `@Validated`, `@NotNull`, `@NotBlank`, `@Size`, `@Pattern`,
and custom `@Constraint` validators across all `source_roots`.
Determine where validation is enforced: controller parameters, request DTOs, service layer, or all.

**Step 3 — DTO and Mapper Scan**
Find request and response DTO/payload classes. Extract naming convention:
`XxxRequest`/`XxxResponse`? `XxxDto`? `XxxPayload`? `XxxCommand`/`XxxView`?

Identify the DTO implementation style:
Lombok `@Data`/`@Value`/`@Builder`? Plain getters/setters? Java records?

Identify the mapping strategy: MapStruct `@Mapper` interfaces? Manual mapper classes?
Inline mapping inside service methods?

**Step 4 — Naming Convention Scan**
Extract domain-specific terminology from class and method names
(e.g., `customerId` vs `userId`? `transaction` vs `payment`?).
Note HTTP method naming patterns on controllers and repository method naming patterns.

**Step 5 — Configuration Property Style Scan**
Search for `@Value("${...}")` usages across all `source_roots`.
Search for `@ConfigurationProperties` classes.
Determine the preferred injection style. If `@ConfigurationProperties` is used, note the
prefix pattern and whether it uses records, `@ConstructorBinding`, or plain setters.
Extract one real example.

**Write your findings to `.claude/conventions.md`:**

```markdown
# Conventions

## Logging [confidence]
[logger declaration style]
[exact syntax with examples at info / debug / warn / error level]

## Validation [confidence]
[where validation is enforced, annotations used with examples]

## DTO Pattern [confidence]
[naming convention, implementation style, mapping strategy with example]

## Naming Conventions [confidence]
[domain terminology, key naming patterns]

## Configuration Property Style [confidence]
[preferred approach: @Value or @ConfigurationProperties]

### Example
[real code snippet from this repo]
```

---

### Sub-Agent 3: Error Handling Agent

**Context required:** Full REPO PROFILE block.

**Your job:** Map the complete exception and error response architecture.

**Step 1 — Global Exception Handler**
Find the `@RestControllerAdvice` or `@ControllerAdvice` class. Read it entirely.
Extract: which exception types are handled, HTTP status each maps to, response body returned.

**Step 2 — Exception Hierarchy**
Find all custom exception classes across all `source_roots`. Build the hierarchy:
base exception → domain-specific subclasses.
Note: error code field? Is it an enum? Checked or unchecked?

**Step 3 — Error Response Model**
Find the error response payload class. Extract its fields: `code`? `message`?
`timestamp`? `details`? `traceId`?
Note: is there a generic response envelope wrapping both success and error payloads?

**Write your findings to `.claude/error-handling.md`:**

```markdown
# Error Handling

## Global Exception Handler
[class name and package]
[table: exception type → HTTP status → response format]

## Exception Hierarchy
[base exception class, domain subclasses, error code enum if present]

## Error Response Model
[class name, fields and their types]

### Example — throwing a domain exception
[real code snippet from this repo]

### Example — error response payload
[real JSON or class snippet]
```

---

### Sub-Agent 4: Integrations & Utilities Agent

**Context required:** Full REPO PROFILE block. Check the `frameworks` field before scanning
— only search for libraries actually listed there. Do not scan for Feign if Kafka is absent, etc.

**Your job:** Capture how this service talks to the outside world and what shared utilities it provides.

**Step 1 — Outbound HTTP Client Scan**
Based on `frameworks` in the REPO PROFILE:
- If Feign/OpenFeign is listed: find all `@FeignClient` interfaces — read them fully.
  Extract: interface declaration style, base URL configuration source, request/response types,
  downstream error handling (`ErrorDecoder`? `fallback`? try-catch in service layer?)
- If Feign is absent, search for `RestTemplate` bean declarations and injection patterns
- If neither, search for `WebClient` builder configuration and usage
- Note which HTTP client pattern this service standardizes on

**Step 2 — Messaging Scan** *(only if Kafka is in `frameworks`)*
- Find `@KafkaListener` annotations — extract topic naming pattern, consumer group pattern,
  message deserialization type
- Find `KafkaTemplate` usages — extract topic naming pattern, message type
- Note: are there dedicated publisher/consumer service classes? Which package?

**Step 3 — Utility Classes Scan**
Find classes in packages named `util`, `utils`, `helper`, `common`, `shared`.
For each: static utility class or Spring bean? One-line purpose summary.
Find custom annotations (not Spring built-ins) — purpose and where applied.
Find constants classes or enums used across multiple packages.

**Write your findings to `.claude/integrations.md`:**

```markdown
# Integrations & Utilities

## Outbound HTTP Client [confidence]
[which client library: Feign / RestTemplate / WebClient]
[how base URLs are configured]
[how downstream errors are handled]

### Example
[real code snippet from this repo]

## Kafka Patterns [confidence]
[omit this section if Kafka is not in frameworks]
[consumer: topic naming, group pattern, deserialization]
[producer: topic naming, message types]

## Utility Classes
[table: class name → type (static/bean) → purpose]

## Custom Annotations
[table: annotation → where applied → purpose]

## Constants & Enums
[shared constants/enum classes and what they represent]
```

---

### Sub-Agent 5: Persistence Agent

**Context required:** Full REPO PROFILE block.

**Your job:** Extract how entities are modeled and persisted in this service.

**Step 1 — Entity Structure Scan**
Find all `@Entity` classes (max 5 across all `source_roots`).
- Is there a base entity class others extend?
- Base entity fields: `id`, `createdAt`, `updatedAt`, `version`, others
- ID generation strategy: `@GeneratedValue(strategy = ...)` — IDENTITY, SEQUENCE, UUID?

**Step 2 — Audit & Soft Delete Scan**
- JPA auditing: `@CreatedDate`, `@LastModifiedDate`, `@CreatedBy`, `@LastModifiedBy`
  — is `@EnableJpaAuditing` present?
- Soft delete: `deletedAt`/`isDeleted` field? `@SQLDelete` or `@Where` annotations?

**Step 3 — Relationship Scan**
Find `@OneToMany`, `@ManyToOne`, `@ManyToMany`, `@OneToOne` usages.
For each: fetch type (LAZY vs EAGER), cascade type, join column naming convention.

**Step 4 — Database Migration Scan**
- If Flyway is in `frameworks`: find migration files under `src/main/resources/db/migration`.
  Note naming convention (`V1__`, `V20240101__`, etc.) and whether SQL or Java-based.
- If Liquibase is in `frameworks`: find changelog files; note format (XML/YAML/SQL).
- Extract: how table names relate to entity names (implicit snake_case? explicit `@Table`?)

**Write your findings to `.claude/persistence.md`:**

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

## Database Migrations [confidence]
[tool: Flyway / Liquibase / none, naming convention, migration format]
```

---

### Sub-Agent 6: Testing Agent

**Context required:** Full REPO PROFILE block.

**Your job:** Extract test conventions so generated tests are indistinguishable from hand-written ones.

**Step 1 — Test Framework Scan**
Confirm JUnit version from build file (JUnit 4 vs 5).
Find which Spring test slices are used: `@SpringBootTest`, `@WebMvcTest`, `@DataJpaTest`,
`@ExtendWith(MockitoExtension.class)`.
If Testcontainers is in `frameworks`: note how it is set up (base test class? per-test?
`@Container` annotation? shared static instance?).
Identify Mockito style: `@Mock`+`@InjectMocks`, `@MockBean`, or BDDMockito (`given/willReturn`).

**Step 2 — Test Naming & Structure Scan**
Find 3–5 test classes under each source root's corresponding `src/test/` path.
- Test class naming convention: `XxxTest`? `XxxSpec`? `XxxShould`?
- Test method naming: `methodName_scenario_expectedResult`? `given_when_then`? `should_...`?
- Is `@DisplayName` used for readable test names?

**Step 3 — Fixture & Assertion Scan**
Find how test data is set up: `@BeforeEach` builders, static factory methods,
`XxxFixture`/`XxxMother` classes, or test builder DSLs?
Assertion library: AssertJ (`assertThat`), Hamcrest, or plain JUnit?

**Write your findings to `.claude/testing.md`:**

```markdown
# Testing

## Framework & Annotations [confidence]
[JUnit version, test slice annotations used, Mockito style]
[Testcontainers setup if present]

## Test Naming Convention [confidence]
[class naming pattern, method naming pattern — with frequency]

## Fixture Strategy [confidence]
[how test data is created — @BeforeEach / builders / fixture classes]

## Assertion Style [confidence]
[AssertJ / Hamcrest / JUnit — with example]
```

---

### Sub-Agent 7: Security Agent

**Context required:** Full REPO PROFILE block.

If `Spring Security` is **not** listed in `frameworks`, write `.claude/security.md` with
a single line: `"No Spring Security configuration found in this service."` and stop.

**Your job:** Extract how this service secures its endpoints and accesses user context.

**Step 1 — Security Config Scan**
Find `SecurityFilterChain` bean — read it entirely.
Extract: which URL patterns are permitted without auth, which require authentication.
Note: is JWT used? Find JWT filter class or `JwtDecoder` bean.

**Step 2 — Method Security Scan**
Search for `@PreAuthorize`, `@PostAuthorize`, `@Secured` usages across all `source_roots`.
Extract: role/permission strings used, naming convention for authorities.

**Step 3 — Principal Access Scan**
Find how the current user is accessed: `SecurityContextHolder.getContext().getAuthentication()`?
A custom `@CurrentUser` annotation? `Principal` method parameter?
Extract the exact pattern with a real example.

**Write your findings to `.claude/security.md`:**

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

After all 7 parallel agents complete, spawn one final agent with these instructions.

**Before doing anything else**, read all 7 context files in this order:
`.claude/architecture.md`, `.claude/conventions.md`, `.claude/error-handling.md`,
`.claude/integrations.md`, `.claude/persistence.md`, `.claude/testing.md`, `.claude/security.md`

Also use the REPO PROFILE (passed in shared context) to know the build tool and which
frameworks are present.

---

### Part A — Extract Real Examples

Before writing any skill file, find the single best real example of each component type
and copy it to the skill's `examples/` directory. These files are the ground truth that
skill files reference — they must come from actual production code, not invented.

**Scoring criteria** (apply to every candidate file):
- +3 — has 3 or more methods
- +2 — uses domain-specific exception types (not generic RuntimeException)
- +2 — uses the service's logging pattern (not System.out)
- +2 — has the full annotation set for its layer
- +1 — uses constructor injection (not @Autowired fields)
- −2 — is a test class or fixture
- −2 — is in a `generated-sources` or `generated` package

Pick the highest-scoring candidate. If tied, prefer the file with more lines.

| Component | Destination |
|-----------|-------------|
| Best REST controller class | `.claude/skills/generate-controller/examples/controller.java` |
| Best Service class | `.claude/skills/generate-service/examples/service.java` |
| Best Repository interface | `.claude/skills/generate-repository/examples/repository.java` |
| Best request DTO + response DTO + mapper | `.claude/skills/generate-dto/examples/dto.java` |
| Best `@Entity` class (prefer one that extends a base entity) | `.claude/skills/generate-entity/examples/entity.java` |
| Best test class (prefer a service unit test) | `.claude/skills/generate-test/examples/test.java` |
| Best HTTP client class/interface *(skip if no outbound HTTP client found)* | `.claude/skills/generate-http-client/examples/http-client.java` |
| The `@RestControllerAdvice` class | `.claude/examples/exception-handler.java` |

Copy files exactly as-is — do not modify or summarize.

---

### Part B — Skill File Template

Every skill is a directory under `.claude/skills/`. The directory contains `SKILL.md`
and an `examples/` subfolder with the real code file from Part A.

```
.claude/skills/generate-<component>/
├── SKILL.md
└── examples/
    └── <file>.java
```

**Required `SKILL.md` format — apply this template for every skill:**

```markdown
---
name: generate-<component>
description: [one sentence — what this skill generates; used for auto-invoke]
---

# [Component] Generation Skill

## Purpose
One sentence: what this skill generates and what failure mode it prevents.

## Scope
- Use for: [specific cases]
- Do NOT use for: [out-of-scope cases]

## Before You Start
1. Read `.claude/avoid.md` — never generate patterns listed there.
2. Read `examples/<file>.java` — this is the real pattern used in this repo.
3. Read the context files referenced in the Rules below.

## Rules

### DO
[8–12 active, specific bullets drawn from the actual artifacts]

### NEVER
[4–6 explicit bans drawn from the actual artifacts]

## Steps
[numbered, one sentence each]
```

Rules for `SKILL.md`:
- `description` is the auto-invoke trigger — one concise sentence; no negations in it
- Put scope limits in the Scope section, not in `description`
- Every DO and NEVER bullet must reflect this repo's actual patterns — not generic advice
- Hard limit: under 150 lines total

---

### Part C — Write Each Skill

Using the template from Part B, write the following skills. The per-skill values below
define what is unique to each skill; all other sections follow the template structure.

---

#### `generate-controller`

```yaml
name: generate-controller
description: Generate a new REST controller class for this Spring Boot service.
```

Context files for Before You Start: `.claude/architecture.md`, `.claude/conventions.md`,
`.claude/error-handling.md`

**DO rules** (fill in from artifacts — these are starting points, not final text):
- Place the controller in the exact package path shown in `.claude/architecture.md`
- Use the class-level annotations seen in `examples/controller.java` — copy them exactly
- Use the logging syntax from `.claude/conventions.md` — no other style
- Delegate all business logic to the service layer
- Return the response type pattern shown in the reference example
- Let exceptions propagate — the global handler in `.claude/error-handling.md` catches them
- Use constructor injection only

**NEVER rules:**
- Catch exceptions inside a controller method — no try/catch blocks here
- Use `System.out.println` or any logger style not in `.claude/conventions.md`
- Add `@Autowired` field injection
- Invent a new response wrapper class not already present in this codebase

**Steps:** Read `examples/controller.java` → identify annotations and class structure →
adapt for the new feature → delegate all logic to the service layer →
verify no try/catch blocks remain

---

#### `generate-service`

```yaml
name: generate-service
description: Generate a new service class for this Spring Boot service.
```

Context files for Before You Start: `.claude/architecture.md`, `.claude/conventions.md`,
`.claude/error-handling.md`

**DO rules:**
- Place `@Transactional` at class-level or method-level exactly as in `.claude/architecture.md`
- Mark read-only methods with `@Transactional(readOnly = true)` if this repo does so
- Use constructor injection only
- Log using the exact syntax in `.claude/conventions.md`
- Throw domain exceptions from the hierarchy in `.claude/error-handling.md`

**NEVER rules:**
- Place `@Transactional` on repository or controller classes
- Catch and swallow exceptions silently
- Use `new RuntimeException()` — always use the domain exception base class
- Add `@Autowired` field injection

**Steps:** Read `examples/service.java` → read `.claude/architecture.md` for transaction
strategy → read `.claude/error-handling.md` for exception hierarchy →
apply correct `@Transactional` placement → use domain exceptions throughout

---

#### `generate-repository`

```yaml
name: generate-repository
description: Generate a new JPA repository interface for this Spring Boot service.
```

Context files for Before You Start: `.claude/persistence.md`

**DO rules:**
- Extend the same base interface shown in `examples/repository.java`
- Follow the method naming convention seen in existing repositories
- Use `@Query` only when a derived method name would be unreadably long
- Use `Pageable` for list operations if the repo does so

**NEVER rules:**
- Mix JPQL and native queries without a clear reason
- Write queries expressible as derived method names
- Annotate the repository with `@Service` or `@Component`

**Steps:** Read `examples/repository.java` → extend the same interface →
name new methods following the same verb pattern (findBy, existsBy, deleteBy) →
add `@Query` only if needed

---

#### `generate-dto`

```yaml
name: generate-dto
description: Generate a request/response DTO and mapper for this Spring Boot service.
```

Context files for Before You Start: `.claude/conventions.md`

**DO rules:**
- Use the naming convention from `.claude/conventions.md`
- Use the same field implementation style as the reference (Lombok, records, or plain getters/setters)
- Place validation annotations on fields exactly as shown in the reference
- Use the same mapping approach (MapStruct interface, manual mapper class, or inline)

**NEVER rules:**
- Invent a new naming pattern for DTOs not already in use
- Add Jackson annotations (`@JsonProperty`) if the repo does not use them
- Put business logic inside a DTO class

**Steps:** Read `examples/dto.java` → name the new DTO following the same convention →
copy the field annotation style → write the mapper in the same location and style

---

#### `generate-entity`

```yaml
name: generate-entity
description: Generate a new JPA entity class for this Spring Boot service.
```

Context files for Before You Start: `.claude/persistence.md`

**DO rules:**
- Extend the same base entity class used across this repo (if one exists)
- Use the ID generation strategy shown in `.claude/persistence.md`
- Include audit fields only if the base entity or `@EnableJpaAuditing` pattern covers them
- Use LAZY fetch type for all `@OneToMany` and `@ManyToMany` by default
- Follow the join column naming convention shown in existing entities
- Apply soft delete pattern if this repo uses it

**NEVER rules:**
- Use EAGER fetch on collection relationships
- Redefine audit fields (`createdAt`, `updatedAt`) that are already in the base entity
- Use `@Table` without following the existing table naming convention
- Generate an entity without inheriting from the base entity if one exists in this repo

**Steps:** Read `.claude/avoid.md` → read `examples/entity.java` →
read `.claude/persistence.md` for base entity, ID strategy, and audit setup →
extend the base entity → add only domain-specific fields →
apply relationship annotations per the detected convention

---

#### `generate-test`

```yaml
name: generate-test
description: Generate a test class for this Spring Boot service.
```

Context files for Before You Start: `.claude/testing.md`

**DO rules:**
- Use the test slice annotation appropriate for the component
  (`@WebMvcTest` for controllers, `@ExtendWith(MockitoExtension.class)` for services)
- Follow the method naming convention from `.claude/testing.md`
- Use the assertion library this repo uses (AssertJ / Hamcrest / JUnit)
- Create test fixtures using the same strategy (builders / `@BeforeEach` / fixture class)
- Mock dependencies using the same style (`@MockBean` vs `@Mock` + `@InjectMocks`)

**NEVER rules:**
- Mix JUnit 4 and JUnit 5 annotations
- Use `@SpringBootTest` for unit tests — only for integration tests if this repo does
- Use `System.out.println` for debugging in tests
- Write tests without assertions

**Steps:** Read `examples/test.java` → read `.claude/testing.md` →
choose the correct test slice for the component → set up fixtures per the repo's strategy →
name the test class and methods following the detected convention

---

#### `generate-http-client` *(only write this skill if an outbound HTTP client is present in `.claude/integrations.md`)*

```yaml
name: generate-http-client
description: Generate an outbound HTTP client for this Spring Boot service.
```

Context files for Before You Start: `.claude/integrations.md`

**DO rules:**
- Use the HTTP client library identified in `.claude/integrations.md`
  (Feign, RestTemplate, or WebClient — whichever this service standardizes on)
- Place the client in the same package as existing clients
- Configure the base URL using the same approach (property key, config class)
- Define request/response types using the same model location pattern
- Handle downstream errors using the same strategy shown in `.claude/integrations.md`

**NEVER rules:**
- Use a different HTTP client library than the one this service standardizes on
- Hardcode URLs — always read from configuration
- Add a `fallback` or `fallbackFactory` unless the reference example uses one
- Mix domain DTOs with client-specific response models without a mapper

**Steps:** Read `examples/http-client.java` → read `.claude/integrations.md` for
error-handling strategy → declare the client following the reference →
wire the URL config using the same key pattern → apply the same error handling

---

### Part D — Write `.claude/avoid.md`

Read all 7 context files. Write `.claude/avoid.md` capturing patterns found in the
codebase that should not be replicated in new code:

```markdown
# Patterns to Avoid

## Deprecated Code Patterns
[patterns found in old files but superseded — e.g., "RestTemplate exists in 2 legacy classes
but has been replaced by Feign — do not use RestTemplate in new code"]

## Wrong Injection Styles
[e.g., "@Autowired field injection exists in 3 old classes — always use constructor injection"]

## Banned Exception Types
[e.g., "do not throw RuntimeException directly — use the domain exception hierarchy"]

## Anti-Patterns Observed
[any other low-frequency / low-confidence patterns that are clearly inconsistent
with the majority convention — note the frequency that makes them anti-patterns]
```

Every skill reads this file before generating. If a pattern appears here, it must not
appear in any generated code regardless of what the user asks.

---

## Phase 4: Write CLAUDE.md and Metadata

**First, write `.claude/metadata.json`** using values from the REPO PROFILE and the
list of skill directories actually created:

```json
{
  "serviceName": "[name from REPO PROFILE]",
  "buildTool": "[maven | gradle]",
  "moduleStructure": "[single | multi]",
  "modules": ["[module names, or empty array]"],
  "generatedAt": "[ISO 8601 timestamp]",
  "commitSha": "[git rev-parse --short HEAD, or 'unknown' if git not available]",
  "scanVersion": "2.0",
  "skillsGenerated": ["[names of skill directories actually written]"]
}
```

**Then write `.claude/CLAUDE.md`** (concise — it loads every session):

```markdown
# [Service Name] — Claude Code Context

## What This Service Does
[1–2 sentences from README or build file description]

## Key Dependencies
[bullet list of most relevant ones]

## Build & Run
[Use the wrapper_cmd and build tool from the REPO PROFILE. Show only the actual tool —
do not list both Maven and Gradle. Use the wrapper if present, bare command otherwise.]
- Build: `[wrapper_cmd] [build-command]`
- Test: `[wrapper_cmd] [test-command]`
- Run: `[wrapper_cmd] [run-command]`

## Architecture Summary
[2–3 sentences: layer structure, module organization, transaction strategy]
→ Full details: `.claude/architecture.md`

## Key Conventions
[2 sentences: logging style, DTO naming, config injection style]
→ Full details: `.claude/conventions.md`

## Error Handling Summary
[1–2 sentences: base exception class, where the global handler lives]
→ Full details: `.claude/error-handling.md`

## Integrations Summary
[1–2 sentences: which HTTP client is used, Kafka if present, key utility classes]
→ Full details: `.claude/integrations.md`

## Persistence Summary
[1–2 sentences: base entity class, ID strategy, soft delete if used, migration tool]
→ Full details: `.claude/persistence.md`

## Testing Summary
[1–2 sentences: JUnit version, mock style, assertion library]
→ Full details: `.claude/testing.md`

## Security Summary
[1 sentence: how endpoints are protected, how current user is accessed]
→ Full details: `.claude/security.md`

## Patterns to Avoid
Before generating any code, read `.claude/avoid.md`.

## Generating New Components

| Task | Skill | Reference Example |
|------|-------|-------------------|
| New REST endpoint or controller | `.claude/skills/generate-controller/SKILL.md` | `examples/controller.java` |
| New service class | `.claude/skills/generate-service/SKILL.md` | `examples/service.java` |
| New repository or query | `.claude/skills/generate-repository/SKILL.md` | `examples/repository.java` |
| New DTO or mapper | `.claude/skills/generate-dto/SKILL.md` | `examples/dto.java` |
| New entity | `.claude/skills/generate-entity/SKILL.md` | `examples/entity.java` |
| New test | `.claude/skills/generate-test/SKILL.md` | `examples/test.java` |
| New HTTP client | `.claude/skills/generate-http-client/SKILL.md` | `examples/http-client.java` |

NEVER introduce patterns, annotations, logging styles, or exception types
that are not already present in this codebase.
```

---

## Done

Report a summary of what was extracted:
- Each `.claude/` file created and its size in lines
- Which optional skill files were written vs skipped (http-client, etc.) and why
- Top 3–5 most distinctive patterns found, each with frequency and confidence
  (e.g., "method-level `@Transactional`: 80% [High]", "`@ConfigurationProperties`: 100% [High]")
- Any patterns added to `avoid.md` and why
- Any gaps where information was absent or ambiguous that would benefit from a future re-scan
