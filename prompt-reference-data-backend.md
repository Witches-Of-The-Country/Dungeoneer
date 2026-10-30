Focus: Backend / Java 21 / Spring Boot 4 / Clean Architecture / PostgreSQL Flyway
Task: Execute Implementation of Change `reference-data-backend` in `player-character-service`

## Execution Context
- Active Plan: `openspec/changes/reference-data-backend/tasks.md`
- Linked Proposal: `openspec/changes/reference-data-backend/proposal.md`
- Delta Spec: `openspec/changes/reference-data-backend/specs/player-character-data-model-delta.md`
- Target Service: `player-character-service/backend`
- Quality & Architecture Contract:
  - Strict Clean Architecture (Hexagonal): domain model MUST have zero framework imports (no Spring, JPA, or Lombok).
  - TDD workflow (Red -> Green -> Refactor): cover happy paths (TP-01..TP-03) and edge cases (TE-01..TE-02).
  - Dual Ruleset Support: cleanly handle `SRD_2014` and `SRD_2024` reference data across schema, seed data, domain, and API.
  - Global Maven: always invoke global `mvn` (e.g., `mvn -f player-character-service/backend/pom.xml ...`), never wrapper.
  - Coverage Gate: JaCoCo line coverage must be ≥ 80% on domain and application layers.
  - Defensive Validation: validate `?ruleset=` query parameter; return RFC 7807 `422 Unprocessable Entity` ProblemDetail for invalid values.
  - Phase Pauses & Commits: execute one phase at a time, verify locally, present the ready-to-copy Conventional Commit command, update the cumulative walkthrough (`openspec/changes/reference-data-backend/walkthrough.md` in pt-BR), and pause for user confirmation before advancing to the next phase.

---

### Phase 1: Flyway Migrations & JPA Entity Schema Alignment

#### Execution Context
- Scope: Execute Phase 1 only. Align database schema and seed authoritative reference data.

#### Implementation Steps
1. Create Flyway migration `V2__align_schema_and_create_reference_tables.sql` in `player-character-service/backend/src/main/resources/db/migration/`:
   - Create schema `player_character` (`CREATE SCHEMA IF NOT EXISTS player_character;`).
   - Move existing tables `public.characters` and `public.ability_scores` to `player_character`.
   - Create tables `player_character.archetypes`, `player_character.lineages`, and `player_character.backgrounds` with:
     - `archetypes`: `id UUID PRIMARY KEY`, `name VARCHAR(50) NOT NULL`, `hit_dice VARCHAR(3) NOT NULL`, `primary_ability VARCHAR(15) NOT NULL`, `saving_throw_proficiencies VARCHAR[] NOT NULL`, `armor_proficiencies VARCHAR[]`, `weapon_proficiencies VARCHAR[]`, `spellcasting_type VARCHAR(15) NOT NULL`, `spellcasting_ability VARCHAR(15)`, `ruleset VARCHAR(10) NOT NULL`, `source VARCHAR(10) NOT NULL DEFAULT 'SRD'`, `owner_id UUID NULL`, `UNIQUE (name, ruleset, source)`.
     - `lineages`: `id UUID PRIMARY KEY`, `name VARCHAR(50) NOT NULL`, `description TEXT`, `speed INT NOT NULL DEFAULT 30`, `size VARCHAR(10) NOT NULL`, `ability_bonuses JSONB NOT NULL DEFAULT '{}'`, `traits_summary TEXT[]`, `ruleset VARCHAR(10) NOT NULL`, `source VARCHAR(10) NOT NULL DEFAULT 'SRD'`, `owner_id UUID NULL`, `UNIQUE (name, ruleset, source)`.
     - `backgrounds`: `id UUID PRIMARY KEY`, `name VARCHAR(50) NOT NULL`, `description TEXT`, `skill_proficiencies VARCHAR[] NOT NULL`, `tool_proficiencies VARCHAR[]`, `languages VARCHAR[]`, `allowed_ability_bonuses VARCHAR[]`, `origin_feat_name VARCHAR(100) NULL`, `origin_feat_description TEXT NULL`, `ruleset VARCHAR(10) NOT NULL`, `source VARCHAR(10) NOT NULL DEFAULT 'SRD'`, `owner_id UUID NULL`, `UNIQUE (name, ruleset, source)`.
   - Add foreign keys on `player_character.characters` (`lineage_id -> lineages.id`, `background_id -> backgrounds.id`).
2. Create Flyway migration `V3__seed_srd_2014_and_2024_reference_data.sql`:
   - Seed all 13 SRD classes (`Barbarian`, `Bard`, `Cleric`, `Druid`, `Fighter`, `Monk`, `Paladin`, `Ranger`, `Rogue`, `Sorcerer`, `Warlock`, `Wizard`, `Artificer`) for both `SRD_2014` and `SRD_2024` (26 rows total).
   - Seed all 9 SRD lineages (`Human`, `Elf`, `Dwarf`, `Halfling`, `Gnome`, `Half-Elf`, `Half-Orc`, `Tiefling`, `Dragonborn`) for both `SRD_2014` (with fixed ASI JSONB) and `SRD_2024` (traits summary, empty `{}` ASI) (18 rows total).
   - Seed all 6 SRD backgrounds (`Acolyte`, `Criminal`, `Folk Hero`, `Noble`, `Sage`, `Soldier`) for both `SRD_2014` and `SRD_2024` (with 3 allowed ASI abilities and origin feat metadata) (12 rows total).
3. Update existing JPA entities in `com.dungeoneer.playerCharacter.adapter.out.persistence`:
   - Update `PlayerCharacterJpaEntity` and `AbilityScoresJpaEntity` `@Table` annotations to include `schema = "player_character"`.

#### Quality Gates & Verification
- Compile and run existing unit/integration tests to ensure schema alignment causes no regressions:
  `mvn -f player-character-service/backend/pom.xml test`
- Ready-to-use commit command:
  ```bash
  git add player-character-service/backend/src/main/resources/db/migration/ player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/out/persistence/
  git commit -m "feat(player-character): add schema alignment, reference tables, and dual-ruleset seed migrations"
  ```

#### Phase Handoff & Cumulative Walkthrough
- Create `openspec/changes/reference-data-backend/walkthrough.md` in Portuguese (pt-BR) recording:
  1. Files created/modified in Phase 1.
  2. Database migration and existing test verification results.
  3. Pause and present the commit command to the user before advancing to Phase 2.

---

### Phase 2: Domain Models & Unit Tests (TDD)

#### Execution Context
- Scope: Execute Phase 2 only. Zero framework imports in `domain/` package.

#### Implementation Steps
1. Implement pure Java domain models in `com.dungeoneer.playerCharacter.domain.model`:
   - `HitDiceEnum`: `D6(6)`, `D8(8)`, `D10(10)`, `D12(12)` with `averagePerLevel()` helper method.
   - `SpellcastingType`: `FULL`, `HALF`, `THIRD`, `PACT`, `NONE`.
   - `Archetype`: rich domain entity with id, name, hitDice, primaryAbility, proficiencies (saving throws, armor, weapons), spellcasting metadata, ruleset, source, ownerId.
   - `Lineage`: rich domain entity with id, name, description, speed, size, abilityBonuses (`Map<String, Integer>` for 2014 ASI), traitsSummary (`List<String>`), ruleset, source, ownerId.
   - `Background`: rich domain entity with id, name, description, proficiencies (skills, tools, languages), allowedAbilityBonuses (`List<String>` for 2024 ASI), originFeatName, originFeatDescription, ruleset, source, ownerId.
   - Enforce domain validation (non-blank name, non-null ruleset, positive speed, immutable collections) and throw domain exceptions on violations.
2. Implement domain unit tests in `src/test/java/com/dungeoneer/playerCharacter/domain/model/`:
   - TP-01: Domain construction and getters for `Archetype`, `Lineage`, `Background`, `HitDiceEnum`.
   - TE-02: Edge case validation tests (blank name, null ruleset, defensive copies).

#### Quality Gates & Verification
- Run domain unit tests:
  `mvn -f player-character-service/backend/pom.xml test -Dtest="com.dungeoneer.playerCharacter.domain.**.*Test"`
- Ready-to-use commit command:
  ```bash
  git add player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/domain/ player-character-service/backend/src/test/java/com/dungeoneer/playerCharacter/domain/
  git commit -m "feat(player-character): implement reference domain models and domain unit tests"
  ```

#### Phase Handoff & Cumulative Walkthrough
- Append to `openspec/changes/reference-data-backend/walkthrough.md` (pt-BR):
  1. Domain models and enums created.
  2. Domain unit test execution results.
  3. Pause and present the commit command to the user before advancing to Phase 3.

---

### Phase 3: Application Layer (Ports, Use Cases & Service)

#### Execution Context
- Scope: Execute Phase 3 only.

#### Implementation Steps
1. Define inbound port in `com.dungeoneer.playerCharacter.application.port.in`:
   - `GetReferenceDataUseCase`: methods for querying classes, lineages, and backgrounds, accepting an optional `Ruleset` filter.
2. Define outbound port in `com.dungeoneer.playerCharacter.application.port.out`:
   - `ReferenceDataRepository`: querying methods for classes, lineages, and backgrounds by ruleset.
3. Implement `ReferenceDataService` in `com.dungeoneer.playerCharacter.application.service`:
   - Implements `GetReferenceDataUseCase`, delegating queries to `ReferenceDataRepository`.
4. Write Mockito unit tests for `ReferenceDataService` in `src/test/java/com/dungeoneer/playerCharacter/application/service/`:
   - Verify behavior with ruleset filter applied and without filter.

#### Quality Gates & Verification
- Run application layer unit tests:
  `mvn -f player-character-service/backend/pom.xml test -Dtest="com.dungeoneer.playerCharacter.application.**.*Test"`
- Ready-to-use commit command:
  ```bash
  git add player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/ player-character-service/backend/src/test/java/com/dungeoneer/playerCharacter/application/
  git commit -m "feat(player-character): implement reference data ports, service, and unit tests"
  ```

#### Phase Handoff & Cumulative Walkthrough
- Append to `openspec/changes/reference-data-backend/walkthrough.md` (pt-BR):
  1. Ports, services, and DTOs implemented.
  2. Application unit test execution results.
  3. Pause and present the commit command to the user before advancing to Phase 4.

---

### Phase 4: Persistence Adapter & Testcontainers Integration Tests

#### Execution Context
- Scope: Execute Phase 4 only.

#### Implementation Steps
1. Implement JPA entities in `com.dungeoneer.playerCharacter.adapter.out.persistence`:
   - `ArchetypeJpaEntity` (`@Table(name = "archetypes", schema = "player_character")`).
   - `LineageJpaEntity` (`@Table(name = "lineages", schema = "player_character")`) with JSONB / converter mapping for `ability_bonuses`.
   - `BackgroundJpaEntity` (`@Table(name = "backgrounds", schema = "player_character")`).
   - Define Spring Data JPA repository interfaces for archetypes, lineages, and backgrounds.
2. Implement `ReferenceDataPersistenceAdapter` implementing `ReferenceDataRepository`:
   - Map JPA entities to pure domain models (`Archetype`, `Lineage`, `Background`).
   - Support querying all entries or filtering by `Ruleset`.
3. Write Testcontainers integration test in `src/test/java/com/dungeoneer/playerCharacter/adapter/out/persistence/`:
   - TP-02: Run against PostgreSQL Testcontainer running Flyway migrations `V1`–`V3`.
   - Verify table counts: 26 classes (13 per ruleset), 18 lineages (9 per ruleset), 12 backgrounds (6 per ruleset).
   - Verify ruleset filtering for both `SRD_2014` and `SRD_2024`.

#### Quality Gates & Verification
- Run persistence integration tests:
  `mvn -f player-character-service/backend/pom.xml test -Dtest="*PersistenceAdapter*Test,*ReferenceData*IT"`
- Ready-to-use commit command:
  ```bash
  git add player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/out/persistence/ player-character-service/backend/src/test/java/com/dungeoneer/playerCharacter/adapter/out/persistence/
  git commit -m "feat(player-character): implement reference persistence adapter and integration tests"
  ```

#### Phase Handoff & Cumulative Walkthrough
- Append to `openspec/changes/reference-data-backend/walkthrough.md` (pt-BR):
  1. JPA entities and persistence adapter created.
  2. Testcontainers integration test results.
  3. Pause and present the commit command to the user before advancing to Phase 5.

---

### Phase 5: Web Adapter, Ruleset Validation & MockMvc API Tests

#### Execution Context
- Scope: Execute Phase 5 only.

#### Implementation Steps
1. Implement Web DTOs and `ReferenceDataController` in `com.dungeoneer.playerCharacter.adapter.in.web`:
   - Endpoints:
     - `GET /api/reference/classes?ruleset=SRD_2014|SRD_2024`
     - `GET /api/reference/lineages?ruleset=SRD_2014|SRD_2024`
     - `GET /api/reference/backgrounds?ruleset=SRD_2014|SRD_2024`
   - DTOs exposing hit dice, proficiencies, ASI metadata (`abilityBonuses` for 2014, `allowedAbilityBonuses` for 2024), and origin feat details.
2. Defensive Validation & RFC 7807 Exception Handling:
   - Validate `ruleset` string against `Ruleset` enum (`SRD_2014`, `SRD_2024`).
   - Ensure invalid `ruleset` values produce `422 Unprocessable Entity` RFC 7807 `ProblemDetail` response via `@ControllerAdvice` / exception handler.
3. Write MockMvc API tests in `src/test/java/com/dungeoneer/playerCharacter/adapter/in/web/`:
   - TP-03: `GET /api/reference/{classes,lineages,backgrounds}` with and without valid `ruleset` parameter returning `200 OK`.
   - TE-01: `GET /api/reference/classes?ruleset=INVALID` returning `422 Unprocessable Entity` ProblemDetail.

#### Quality Gates & Verification
- Run web layer tests:
  `mvn -f player-character-service/backend/pom.xml test -Dtest="*ReferenceDataController*Test"`
- Ready-to-use commit command:
  ```bash
  git add player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/in/web/ player-character-service/backend/src/test/java/com/dungeoneer/playerCharacter/adapter/in/web/
  git commit -m "feat(player-character): implement reference REST controller, validation, and web tests"
  ```

#### Phase Handoff & Cumulative Walkthrough
- Append to `openspec/changes/reference-data-backend/walkthrough.md` (pt-BR):
  1. Web controller, DTOs, and exception handler updates.
  2. MockMvc test execution results.
  3. Pause and present the commit command to the user before advancing to Phase 6.

---

### Phase 6: Full Verification, Spec Delta & Final Consolidation

#### Execution Context
- Scope: Execute Phase 6. Complete test coverage verification, update spec deltas, and prepare for archival.

#### Implementation Steps
1. Run full build and verify JaCoCo coverage:
   - Verify JaCoCo line coverage is ≥ 80% on domain and application layers:
     `mvn -f player-character-service/backend/pom.xml clean verify`
2. Update specification delta in `openspec/changes/reference-data-backend/specs/player-character-data-model-delta.md`:
   - Ensure all reference data schema changes and ruleset mechanics are accurately recorded.
3. Update `openspec/changes/reference-data-backend/tasks.md`:
   - Mark all completed tasks with `[x]`.
4. Finalize `openspec/changes/reference-data-backend/walkthrough.md` in Portuguese (pt-BR):
   - Consolidate all changes across Phases 1 through 6.
   - Document test verification and coverage summary.
   - Provide clear actionable next steps (e.g. frontend wizard integration or archival via `sdd-spec-archive`).

#### Quality Gates & Verification
- Run full Maven build:
  `mvn -f player-character-service/backend/pom.xml clean verify`
- Ready-to-use commit command:
  ```bash
  git add openspec/changes/reference-data-backend/ prompt-reference-data-backend.md
  git commit -m "chore(player-character): complete verification, delta spec documentation, and walkthrough"
  ```

#### Final Phase Handoff
- Present the consolidated walkthrough to the user and notify readiness for `sdd-spec-validate` or `sdd-spec-archive`.
