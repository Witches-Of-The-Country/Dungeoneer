# Proposal: Migrate Legacy Specs To Openspec

> **Change ID:** `migrate-legacy-specs-to-openspec`
> **Status:** approved
> **Author:** @jaca
> **Created:** 2026-10-29
> **Last Updated:** 2026-10-29

---

## Summary

Migrate the existing Dungeoneer documentation and specifications from `docs/` (`docs/spec/SPEC.md`, `docs/spec/res/*`, `docs/USE-CASES.md`, `docs/ARCHITECTURE.md`, and `docs/STATE.md`) into the canonical Spec-Driven Development (`openspec/`) model. Populate `openspec/specs/sdd-stack-contract.md` with Dungeoneer's polyglot microservices stack (Java 21 / Spring Boot 4.x and TypeScript 5 / Next.js 16), consolidate domain and architectural specs into `openspec/specs/`, update `AGENTS.md` to reference the new canonical living specs, and remove superseded legacy files in `docs/`.

## Motivation

Dungeoneer is adopting the `sdd-scaffold-plugin` (OpenSpec) workflow, which requires all canonical living specifications to reside in `openspec/specs/` with structured YAML frontmatter (`id`, `title`, `version`, `status`, `owners`, `scope.paths`, `resources`) and standardized sections (`Context & Objectives`, `Functional Requirements`, `Non-Functional Requirements`, `Out of Scope`, `Revision History`), while active changes live in `openspec/changes/`. Currently, specifications are fragmented across `docs/spec/SPEC.md`, `docs/spec/res/*.md`, `docs/USE-CASES.md`, `docs/ARCHITECTURE.md`, and `docs/STATE.md`, and `openspec/specs/sdd-stack-contract.md` contains unpopulated template placeholders.

## Scope

### In Scope
- Populate `openspec/specs/sdd-stack-contract.md` (via `sdd-stack-contract-delta.md`) with full backend (Java 21, Spring Boot 4.x, Maven, JUnit 5, Testcontainers, JaCoCo, PostgreSQL 15, Flyway) and frontend (Next.js 16, React 19, TypeScript 5, TailwindCSS 4, Vitest, Playwright) stack contract details.
- Create 6 canonical living specifications (staged in `openspec/changes/migrate-legacy-specs-to-openspec/specs/` and consolidated into `openspec/specs/`):
  1. `system-architecture.md` (from `docs/spec/SPEC.md`, `docs/spec/res/architecture-diagram.md`, `docs/ARCHITECTURE.md`)
  2. `player-character-domain.md` (from `README.md` domain patterns, `docs/USE-CASES.md` UC1–UC5, `docs/spec/SPEC.md`, `docs/spec/res/api-contracts.md`)
  3. `player-character-data-model.md` (from `docs/spec/res/data-models.md`, `docs/spec/res/er-diagram.md`)
  4. `campaign-domain.md` (from `docs/spec/SPEC.md` Phase 2/3, `docs/USE-CASES.md` UC6–UC7, `docs/spec/res/er-diagram.md`, `docs/spec/res/api-contracts.md`)
  5. `frontend-web-app.md` (from `docs/spec/res/frontend-spec.md`, `docs/STATE.md`)
  6. `infrastructure-and-testing.md` (from `docs/spec/res/infrastructure.md`, `docs/spec/res/testing-strategy.md`)
- Update `AGENTS.md` ("Specifications & Knowledge Index" table) and `README.md` to point to the new canonical specifications in `openspec/specs/`.
- Remove legacy specification files from `docs/` (`docs/spec/SPEC.md`, `docs/spec/res/*`, `docs/ARCHITECTURE.md`, `docs/USE-CASES.md`, `docs/STATE.md`) and remove the `_example` placeholder directory (`openspec/changes/_example/`).

### Out of Scope
- Modifying any production Java or TypeScript source code, database migrations, or CI/CD workflows.
- Implementing unfinished Phase 1 features (such as `PlayerCharacterController` GREEN phase or Character Creation Wizard UI).

## Proposed Solution

### Approach
We will structure Dungeoneer's technical and domain documentation into modular, domain-aligned living specifications following `openspec/specs/_template.md`. First, `sdd-stack-contract-delta.md` will define all build, test, coverage, database, and architectural invariants for `openspec/specs/sdd-stack-contract.md`. Second, six new canonical specifications will capture every requirement, Mermaid diagram (C4, ER, Class, Use Case), table definition, REST endpoint, RFC 7807 error mapping, and UI token from `docs/` without any loss of fidelity.

Once the specifications are authored and validated, `AGENTS.md` and `README.md` will be updated so all AI agents and developers load context directly from `openspec/specs/`, and the legacy `docs/` files will be cleanly removed to prevent dual-source-of-truth drift.

### Impacted Files
<!-- Populated by sdd-code-explorer subagent. -->
| File | Change Type | Notes |
|------|-------------|-------|
| `openspec/specs/sdd-stack-contract.md` | Modify | Populate stack identity, structure, test runners, API contract, DB, and hexagonal rules |
| `openspec/specs/system-architecture.md` | Create | Canonical living spec for system topology, C4 diagrams, Hexagonal layers, and Dual Ruleset strategy |
| `openspec/specs/player-character-domain.md` | Create | Canonical living spec for character domain models, use cases, SRD scope, and REST API contracts |
| `openspec/specs/player-character-data-model.md` | Create | Canonical living spec for `player_character` PostgreSQL schema, ER diagram, and Flyway rules |
| `openspec/specs/campaign-domain.md` | Create | Canonical living spec (draft) for Phase 2/3 campaign, NPC, and encounter management |
| `openspec/specs/frontend-web-app.md` | Create | Canonical living spec for Next.js 16 routes, Dark Fantasy UI tokens, Wizard, and Sheet Dashboard |
| `openspec/specs/infrastructure-and-testing.md` | Create | Canonical living spec for Docker Compose, CI/CD, deployment, TDD, and JaCoCo strategy |
| `AGENTS.md` | Modify | Update Specifications & Knowledge Index table to point to `openspec/specs/*.md` |
| `README.md` | Modify | Update legacy links (`docs/USE-CASES.md`, `docs/ARCHITECTURE.md`) to canonical `openspec/specs/` files |
| `docs/spec/SPEC.md` | Delete | Superseded by `openspec/specs/system-architecture.md` and `sdd-stack-contract.md` |
| `docs/spec/res/architecture-diagram.md` | Delete | Migrated to `openspec/specs/system-architecture.md` |
| `docs/spec/res/er-diagram.md` | Delete | Migrated to `player-character-data-model.md` and `campaign-domain.md` |
| `docs/spec/res/data-models.md` | Delete | Migrated to `openspec/specs/player-character-data-model.md` |
| `docs/spec/res/api-contracts.md` | Delete | Migrated to `player-character-domain.md` and `campaign-domain.md` |
| `docs/spec/res/frontend-spec.md` | Delete | Migrated to `openspec/specs/frontend-web-app.md` |
| `docs/spec/res/infrastructure.md` | Delete | Migrated to `openspec/specs/infrastructure-and-testing.md` |
| `docs/spec/res/testing-strategy.md` | Delete | Migrated to `openspec/specs/infrastructure-and-testing.md` |
| `docs/USE-CASES.md` | Delete | Migrated to `player-character-domain.md` and `campaign-domain.md` |
| `docs/ARCHITECTURE.md` | Delete | Migrated to `openspec/specs/system-architecture.md` |
| `docs/STATE.md` | Delete | Superseded by `openspec/changes/` tracking and living spec statuses |
| `openspec/changes/_example/` | Delete | Remove initial scaffold example change directory |

### Data Model Changes (if applicable)
N/A — This change only migrates specification and documentation files; no database schemas or application DTOs are modified.

## Acceptance Criteria

- [x] **Given** the `openspec/specs/sdd-stack-contract.md` file **When** inspected by an agent or developer **Then** all stack identity, project structure, test runner (`mvn clean test`, `npx vitest run`, `npx playwright test`), coverage (`mvn clean verify` ≥80% JaCoCo), API contract, and PostgreSQL 15 + Flyway fields are completely populated with zero placeholder comments remaining.
- [x] **Given** the legacy specs in `docs/` **When** the migration is completed **Then** 6 canonical living specifications exist in `openspec/specs/` (`system-architecture.md`, `player-character-domain.md`, `player-character-data-model.md`, `campaign-domain.md`, `frontend-web-app.md`, `infrastructure-and-testing.md`) with valid YAML frontmatter and all diagrams, tables, and requirements preserved.
- [x] **Given** `AGENTS.md` in the workspace root **When** reading the "Specifications & Knowledge Index" table **Then** all document links point to the canonical `openspec/specs/*.md` files and no broken links to `docs/spec/` remain.
- [x] **Given** the workspace directory tree **When** the migration is finalized **Then** the superseded files in `docs/` (`docs/spec/`, `docs/USE-CASES.md`, `docs/ARCHITECTURE.md`, `docs/STATE.md`) and `openspec/changes/_example/` are removed.

## Defensive Security & Validation Considerations

### Input Validation
- N/A for runtime code (documentation-only migration). All API specifications migrated into `player-character-domain.md` explicitly mandate Jakarta Validation (`@Valid`, `@NotBlank`, `@NotNull`, `@Min`, `@Max`) on all inbound DTOs.

### Secret Isolation
- All migrated infrastructure and stack specs strictly preserve environment variable injection (`${DB_PASSWORD}`, `${JWT_SECRET}`) and prohibit hardcoded credentials in documentation examples or configuration files.

### Authorization & Authentication Impact
- N/A — This change does not modify authentication or authorization code (`SecurityConfig` or `auth-service`). Security specifications in `system-architecture.md` document both the MVP `permitAll()` posture and `auth-service` integration boundaries.

### Other Security Risks
- None — Pure Markdown specification refactoring.

## Testing Strategy & Shift-Left Plan

### Happy Path Tests
| Test ID | Scenario | Test Type | Expected Result |
|---------|----------|-----------|-----------------|
| TP-01 | Verify all 7 canonical files exist in `openspec/specs/` with valid frontmatter and zero template placeholders | Verification / Audit | All 7 files present and formatted according to `_template.md` and `sdd-stack-contract.md` |
| TP-02 | Verify `AGENTS.md` knowledge index links resolve to existing files in `openspec/specs/` | Verification / Audit | 100% of referenced spec paths exist |

### Edge Case & Failure Mode Tests
| Test ID | Scenario | Test Type | Expected Result |
|---------|----------|-----------|-----------------|
| TE-01 | Search repository for dangling references to `docs/spec/` in `AGENTS.md` or `.agents/skills/` | Verification / Grep | Zero broken references in agent rules and skills |
| TE-02 | Ensure existing backend unit tests still pass unaffected | Unit (`mvn clean test`) | Build succeeds with 0 failures |

### Test Commands
- **Run backend tests:** `mvn -f player-character-service/backend/pom.xml clean test`
- **Verify markdown structure:** `Get-ChildItem openspec/specs/`

## Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Loss of detailed table schemas or Mermaid diagrams during consolidation | Low | High | Preserve 100% of Mermaid blocks (C4, ER, Class, Use Case) and SQL column tables verbatim inside the corresponding living specs |
| Agent skills referencing old `docs/spec/` paths | Medium | Medium | Scan `.agents/skills/` and `AGENTS.md` via `sdd-code-explorer` to update any path references |

## Specification Changes & Delta Stubs

### `specs/sdd-stack-contract-delta.md`
- **Target Canonical Spec:** `openspec/specs/sdd-stack-contract.md`
- **Summary of Updates:** Populate complete Dungeoneer stack contract (Java 21, Spring Boot 4.x, Next.js 16, Maven, npm, JUnit 5, Testcontainers, Vitest, Playwright, PostgreSQL 15, Flyway, Clean Architecture rules).

### `specs/system-architecture.md`
- **Target Canonical Spec:** `New Specification` (`openspec/specs/system-architecture.md`)
- **Summary of Updates:** Consolidate system vision, C4 diagrams, Hexagonal architecture, Dual Ruleset engine, data ownership, security, and localization rules.

### `specs/player-character-domain.md`
- **Target Canonical Spec:** `New Specification` (`openspec/specs/player-character-domain.md`)
- **Summary of Updates:** Consolidate domain class diagram, GoF patterns, UC1–UC5, SRD seed scope, REST API endpoints (`/api/characters`, `/api/spells`, `/api/reference`), and RFC 7807 error contracts.

### `specs/player-character-data-model.md`
- **Target Canonical Spec:** `New Specification` (`openspec/specs/player-character-data-model.md`)
- **Summary of Updates:** Consolidate `player_character` ER diagram and all 13 PostgreSQL table definitions and constraints.

### `specs/campaign-domain.md`
- **Target Canonical Spec:** `New Specification` (`openspec/specs/campaign-domain.md`)
- **Summary of Updates:** Consolidate Phase 2/3 Campaign, NPC, and Encounter requirements, `campaign` ER diagram, and planned REST endpoints.

### `specs/frontend-web-app.md`
- **Target Canonical Spec:** `New Specification` (`openspec/specs/frontend-web-app.md`)
- **Summary of Updates:** Consolidate Next.js 16 routes, Dark Fantasy design system tokens, Character Creation Wizard, Character Sheet Dashboard, and state management rules.

### `specs/infrastructure-and-testing.md`
- **Target Canonical Spec:** `New Specification` (`openspec/specs/infrastructure-and-testing.md`)
- **Summary of Updates:** Consolidate Docker Compose topology, CI/CD pipelines, deployment architecture, TDD workflow, and JaCoCo coverage enforcement.

## References

- Agent Guidelines: `AGENTS.md`
