# explore-service Design Spec
**Date:** 2026-06-29

## Problem

Developers work across 10–15 Spring Boot microservices. Each service has different coding
conventions, architecture patterns, logging syntax, and error handling. When Claude Code
generates code, it produces generic output that doesn't match the service's style — wrong
logging format, wrong exception types, wrong DTO naming, wrong transaction placement.
Developers waste time manually correcting generated code to fit the repo.

## Goal

Run one command in any Spring Boot service repo. It extracts the service's coding DNA and
generates a `.claude/` folder. Every future Claude Code session in that repo automatically
inherits the correct context and generates service-native code.

## Constraints

- Zero friction for the developer using the generated context (no manual steps per session)
- Works with Claude Code only for now — designed to extend to other AI tools later
- No external CLI to install or distribute — implemented entirely as a Claude Code custom command
- Developers manually update the `.claude/` folder as the service evolves

## Architecture

### Command

A user-level global custom command stored at `~/.claude/commands/explore-service.md`.
Installed once per machine. Invoked as `/explore-service` from inside Claude Code.

### Execution Model: Orchestrator + Parallel Sub-Agents

```
/explore-service invoked
        │
        ▼
Orchestrator reads repo surface
(pom.xml, package tree, README)
        │
        ▼
Spawns 3 sub-agents in parallel
┌──────────────┬─────────────────┬──────────────────┐
│ Architecture │  Conventions    │ Error Handling   │
│    Agent     │    Agent        │    Agent         │
└──────┬───────┴────────┬────────┴──────────┬───────┘
       ▼                ▼                   ▼
.claude/         .claude/          .claude/
architecture.md  conventions.md    error-handling.md
       │                │                   │
       └────────────────┴──────────┬────────┘
                                   ▼
                         Skills Generator Agent
                         writes .claude/skills/
                                   │
                                   ▼
                         Orchestrator writes CLAUDE.md
```

## Cross-Cutting Agent Rules

All agents apply these rules:
- **Scan exclusions:** skip `target/`, `build/`, `.git/`, `.idea/`, `generated-sources/`, `node_modules/`, `.class`, `.jar`
- **Sampling limits:** max 5 files per layer, max 20 files per concern; prefer richer files
- **Frequency reporting:** each pattern includes a % frequency across scanned files
- **Confidence levels:** `[High]` 5+ locations, `[Medium]` 2–4, `[Low]` 1 or inferred

## Sub-Agent Responsibilities

### Architecture Agent

Extracts:
- Package layer names and module organization strategy (by layer vs by feature)
- Spring annotation usage per layer
- Dependency direction between layers
- **Transaction Strategy**: where `@Transactional` is placed (class-level vs method-level in
  Service layer), whether read-only operations use `@Transactional(readOnly = true)`

Writes: `.claude/architecture.md`

### Conventions Agent

Extracts:
- Logging: exact syntax at all log levels (SLF4J structured? Lombok @Slf4j? key-value pairs?)
- Validation: where validation is enforced and which annotations are used
- DTOs: naming pattern (`XxxRequest`/`XxxResponse`? `XxxDto`?) and mapping strategy
  (MapStruct, manual mappers, inline)
- Naming: domain terminology, field naming, method naming patterns
- **Configuration Property Style**: whether service uses `@Value("${...}")` for direct property
  injection or `@ConfigurationProperties` type-safe beans

Writes: `.claude/conventions.md`

### Error Handling Agent

Extracts:
- `@RestControllerAdvice` / `@ControllerAdvice` class — reads it fully
- Custom exception hierarchy (base class → domain subclasses)
- Error response payload model fields
- HTTP status mapping strategy

Writes: `.claude/error-handling.md`

### Persistence Agent

Extracts entity modeling patterns: base entity inheritance, ID generation, audit fields,
soft delete, and relationship conventions. Writes `.claude/persistence.md`.

### Testing Agent

Extracts test conventions: JUnit version, test slice annotations, Mockito style, naming
convention, fixture strategy, assertion library. Writes `.claude/testing.md`.

### Security Agent

Extracts security config: SecurityFilterChain rules, JWT strategy, method-level security
annotations, and how current user context is accessed. Writes `.claude/security.md`.

### Skills Generator Agent

Reads all three artifacts and writes four skill files that serve as step-by-step
recipes for generating new components:

Part A extracts real examples using a scoring system (richness of annotations, method count,
use of domain exceptions). Part B writes skill files under 150 lines each. Part C writes
`.claude/avoid.md` capturing deprecated or anti-patterns found in the codebase.

All skills open with "read `avoid.md` before generating."

| File | Purpose |
|------|---------|
| `.claude/skills/generate-controller.md` | New REST controller recipe |
| `.claude/skills/generate-service.md` | New service class recipe |
| `.claude/skills/generate-repository.md` | New repository recipe |
| `.claude/skills/generate-dto.md` | New DTO + mapper recipe |
| `.claude/skills/generate-entity.md` | New JPA entity recipe |
| `.claude/skills/generate-test.md` | New test class recipe |

Each skill file contains the pattern in plain language + real code snippets pulled from
the actual codebase as examples.

## Generated File Structure

```
repo-root/
└── .claude/
    ├── CLAUDE.md                    ← auto-loaded by Claude Code; no repo root clutter
    ├── metadata.json                ← serviceName, generatedAt, commitSha, scanVersion
    ├── architecture.md
    ├── conventions.md
    ├── error-handling.md
    ├── integrations.md
    ├── persistence.md
    ├── testing.md
    ├── security.md
    ├── avoid.md                     ← deprecated/anti-patterns — all skills read this first
    ├── examples/
    │   └── exception-handler.java   ← global; referenced by error-handling.md
    └── skills/                      ← each skill is a directory, not a flat .md
        ├── generate-controller/
        │   ├── SKILL.md             ← frontmatter + job/rules/steps; < 150 lines
        │   └── examples/
        │       └── controller.java  ← scored: richest real controller in this repo
        ├── generate-service/
        │   ├── SKILL.md
        │   └── examples/
        │       └── service.java
        ├── generate-repository/
        │   ├── SKILL.md
        │   └── examples/
        │       └── repository.java
        ├── generate-dto/
        │   ├── SKILL.md
        │   └── examples/
        │       └── dto.java
        ├── generate-entity/
        │   ├── SKILL.md
        │   └── examples/
        │       └── entity.java
        ├── generate-test/
        │   ├── SKILL.md
        │   └── examples/
        │       └── test.java
        └── generate-feign-client/   ← only if Feign/HTTP client present
            ├── SKILL.md
            └── examples/
                └── feign-client.java
```

### Skill File Design Principles

Each skill is a directory containing `SKILL.md` and an `examples/` subfolder. `SKILL.md` follows a strict anatomy:
- **YAML frontmatter** — `name` and `description` fields; description lists trigger phrases and what NOT to use the skill for
- **Job** — one sentence describing what failure mode it prevents
- **Before You Start** — read `avoid.md` + read `examples/<file>.java`
- **Rules** — DO (8–12 bullets) and NEVER (4–6 explicit bans)
- **Steps** — numbered, one sentence each
- **Hard limit** — under 150 lines total

Each skill's `examples/` contains the single best real code file for that component type, copied exactly from production code. Skills reference it as `examples/<file>.java` (relative). This keeps skills self-contained and examples up to date.

## Dynamic Runtime Behavior

1. Developer opens Claude Code in any session in the repo
2. Claude auto-loads `CLAUDE.md` — gets service overview and key conventions at a glance
3. Developer asks for a feature or component
4. `CLAUDE.md` instructs Claude to read the relevant skill from `.claude/skills/`
5. Claude generates code following the exact patterns extracted from that repo

No developer action required beyond the initial `/explore-service` run.

## What the Output Enables

- Correct logging syntax on every generated file (no generic `System.out.println`)
- Correct exception types and propagation (no invented RuntimeException subclasses)
- Correct DTO naming and mapping strategy
- Correct `@Transactional` placement and `readOnly` usage
- Correct config injection style (`@Value` vs `@ConfigurationProperties`)

## Out of Scope (Phase 1)

- Kafka / event patterns
- Testing pattern extraction
- Anti-pattern documentation
- Integration with Copilot, Codex, or other AI tools
- Incremental update command (`/update-service`)
- CI/CD automation of re-scanning

## Future Extensions

- `/update-service` command for incremental re-scan when codebase changes significantly
- Additional agents: Kafka patterns, testing conventions, security patterns
- Multi-tool output: generate `.github/copilot-instructions.md` alongside `CLAUDE.md`
- Team sharing: publish `.claude/` artifacts to a central repo for cross-service discovery
