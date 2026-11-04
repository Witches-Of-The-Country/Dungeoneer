Focus: Backend / Java 21 / Spring Boot 4 / Clean Architecture / PostgreSQL Flyway
Task: Execute Implementation of Change `character-crud-backend` in `player-character-service`

## Execution Context
- Active Plan: `openspec/changes/character-crud-backend/tasks.md`
- Linked Proposal: `openspec/changes/character-crud-backend/proposal.md`
- Delta Specs:
  - `openspec/changes/character-crud-backend/specs/player-character-domain-delta.md`
  - `openspec/changes/character-crud-backend/specs/player-character-data-model-delta.md`
- Target Service: `player-character-service/backend`
- Quality & Architecture Contract:
  - Strict Clean Architecture (Hexagonal): domain model MUST have zero framework imports (no Spring, JPA, or Lombok).
  - TDD Workflow (Red -> Green -> Refactor): simultaneously implement unit and integration test suites covering happy paths (TP-01..TP-03) and edge cases/failure modes (TE-01..TE-03).
  - Single-Chance Server-Side Rolls: enforce idempotent generation for ability scores (`4d6kh3`) and HP rolls; verify unused status on creation and consume rolls atomically.
  - Dual Ruleset Support: apply lineage ASI for `SRD_2014` and background ASI (`+2/+1` or `+1/+1/+1` from allowed list) for `SRD_2024`.
  - IDOR / BOLA Prevention: enforce `X-User-Id` header on all `/api/characters/**` endpoints; scope all repository operations by `userId`; return RFC 7807 `404 Not Found` ProblemDetail if an entity belongs to another tenant.
  - Global Maven: always invoke global `mvn` (e.g., `mvn -f player-character-service/backend/pom.xml ...`), never wrapper.
  - Coverage Gate: JaCoCo line coverage must be >= 80% on domain and application layers.
  - Phase Pauses & Commits: execute one phase at a time, verify locally, present the ready-to-copy Conventional Commit command, update the cumulative walkthrough (`openspec/changes/character-crud-backend/walkthrough.md` in pt-BR), and pause for user approval before moving to the next phase.

---

### Phase 1: Database Migration & Dormant Stub Removal

#### Execution Context
- Scope: Execute Phase 1 only. Align database schema for character CRUD and roll persistence, and remove dormant stub artifacts.

#### Implementation Steps
1. Create Flyway migration `V4__add_character_crud_and_roll_tables.sql` in `player-character-service/backend/src/main/resources/db/migration/`:
   - Alter `player_character.characters`:
     - Add `notes TEXT NOT NULL DEFAULT ''`.
     - Add `hp_calculation_method VARCHAR(15) NOT NULL DEFAULT 'MAX'`.
     - Add `has_pending_level_choices BOOLEAN NOT NULL DEFAULT FALSE`.
     - Enforce `user_id UUID NOT NULL`.
   - Create table `player_character.class_progressions`:
     - Columns: `id UUID PRIMARY KEY`, `character_id UUID NOT NULL REFERENCES player_character.characters(id) ON DELETE CASCADE`, `archetype_id UUID NOT NULL REFERENCES player_character.archetypes(id)`, `subclass_id UUID NULL`, `class_level INT NOT NULL CHECK (class_level >= 1 AND class_level <= 20)`.
     - Constraint: `UNIQUE (character_id, archetype_id)`.
   - Create table `player_character.ability_rolls`:
     - Columns: `id UUID PRIMARY KEY`, `user_id UUID NOT NULL`, `rolled_scores INT[] NOT NULL`, `used BOOLEAN NOT NULL DEFAULT FALSE`, `created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()`.
     - Partial index: `CREATE UNIQUE INDEX idx_unused_ability_roll ON player_character.ability_rolls(user_id) WHERE used = FALSE;`.
   - Create table `player_character.hp_rolls`:
     - Columns: `id UUID PRIMARY KEY`, `user_id UUID NOT NULL`, `archetype_id UUID NOT NULL REFERENCES player_character.archetypes(id)`, `target_level INT NOT NULL CHECK (target_level >= 2 AND target_level <= 20)`, `rolled_hp_per_level INT[] NOT NULL`, `used BOOLEAN NOT NULL DEFAULT FALSE`, `created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()`.
     - Partial index: `CREATE UNIQUE INDEX idx_unused_hp_roll ON player_character.hp_rolls(user_id, archetype_id, target_level) WHERE used = FALSE;`.
2. Clean up duplicate dormant stubs:
   - Safely remove unused stub classes: `CreatePlayerCharacterUseCase`, `CharacterRepositoryPort`, `CreatePlayerCharacterService`, and `PlayerCharacterUseCaseConfig`.
   - Ensure clean compilation without orphaned references.

#### Quality Gates & Verification
- Compile project and run existing test suite:
  `mvn -f player-character-service/backend/pom.xml test`
- Ready-to-use commit command:
  ```bash
  git add player-character-service/backend/src/main/resources/db/migration/ player-character-service/backend/src/main/java/
  git commit -m "feat(player-character): add V4 migration for character crud, rolls, class progression and remove dormant stubs"
  ```

#### Phase Handoff & Cumulative Walkthrough
- Create `openspec/changes/character-crud-backend/walkthrough.md` in Portuguese (pt-BR) recording:
  1. Files created/modified during Phase 1.
  2. Database migration and compilation verification results.
  3. Pause and present the commit command to the user before advancing to Phase 2.

---

### Phase 2: Pure Domain Layer & Unit Tests (TDD)

#### Execution Context
- Scope: Execute Phase 2 only. Zero framework imports in `com.dungeoneer.playerCharacter.domain` (no Spring, JPA, or Lombok).

#### Implementation Steps
1. Implement pure domain entities and value objects in `com.dungeoneer.playerCharacter.domain.model`:
   - `ClassProgression`: represents class association and level progression (level 1-20), optional subclass.
   - `AbilityScoreValidator`: validates base ability scores:
     - `STANDARD_ARRAY`: exact permutation of `[15, 14, 13, 12, 10, 8]`.
     - `POINT_BUY`: each score within `8..15`, cost mapping (`8:0, 9:1, 10:2, 11:3, 12:4, 13:5, 14:7, 15:9`), total cost exactly 27 points.
     - `ROLLED`: exact multiset permutation matching the 6 values of an unused ability roll token.
   - `RulesEngine` interface and implementations (`Srd2014RulesEngine`, `Srd2024RulesEngine`):
     - `SRD_2014`: applies fixed lineage ASI bonuses (plus choices for lineages like Half-Elf).
     - `SRD_2024`: validates bonus distribution (`+2/+1` across 2 abilities or `+1/+1/+1` across 3 abilities) against `Background.allowedAbilityBonuses`, capping at 20.
     - Server-side HP calculation:
       - Level 1: `hitDice.max + CON_mod` (minimum 1).
       - Levels `2..level`:
         - `MAX`: `hitDice.max + CON_mod` per level.
         - `AVERAGE`: `(hitDice.max / 2 + 1) + CON_mod` per level.
         - `ROLLED`: sum of `rolled_hp_per_level + CON_mod` per level.
     - Derived attributes: Armour Class (base 10 + DEX mod), Initiative modifier (DEX mod), Proficiency Bonus (based on total level), and Speed (from Lineage).
   - Update `PlayerCharacter` aggregate root:
     - Add fields: `notes`, `hpCalculationMethod`, `hasPendingLevelChoices`, and list of `ClassProgression`.
     - Provide `updateSheet(String name, int hitPointsCurrent, int hitPointsTemp, String notes)` guarding immutable fields (level, ability scores, lineage, background cannot change via sheet edit).
   - Domain exceptions in `com.dungeoneer.playerCharacter.domain.exception`:
     - `CharacterValidationException`, `CharacterNotFoundException`, `RollConflictException`, `InvalidRollTokenException`.
2. Write comprehensive domain unit tests in `src/test/java/com/dungeoneer/playerCharacter/domain/`:
   - TP-01: `AbilityScoreValidatorTest` for standard array permutations, Point Buy exact 27-point cost calculation, and multiset rolled permutation verification.
   - TP-01 & TE-03: `RulesEngineTest` for 2014 lineage ASI, 2024 background ASI validation/capping, and HP calculations (`MAX`, `AVERAGE`, `ROLLED`).
   - `PlayerCharacterTest`: sheet updates and domain invariant rules.

#### Quality Gates & Verification
- Run domain unit test suite:
  `mvn -f player-character-service/backend/pom.xml test -Dtest="com.dungeoneer.playerCharacter.domain.**.*Test"`
- Ready-to-use commit command:
  ```bash
  git add player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/domain/ player-character-service/backend/src/test/java/com/dungeoneer/playerCharacter/domain/
  git commit -m "feat(player-character): implement domain rules engine, ability score validator, progression and domain unit tests"
  ```

#### Phase Handoff & Cumulative Walkthrough
- Append to `openspec/changes/character-crud-backend/walkthrough.md` (pt-BR):
  1. Domain models, value objects, and exceptions implemented.
  2. Domain unit test execution results.
  3. Pause and present the commit command to the user before advancing to Phase 3.

---

### Phase 3: Application Layer (Ports, Use Cases, DTOs & Services)

#### Execution Context
- Scope: Execute Phase 3 only.

#### Implementation Steps
1. Define Inbound Ports in `com.dungeoneer.playerCharacter.application.port.in`:
   - `CreateCharacterUseCase`: handles character creation with ability score validation, ASI application, HP computation, and roll consumption.
   - `GetCharacterUseCase`: retrieves a character sheet by ID and user ID.
   - `ListCharactersUseCase`: retrieves all character summaries owned by user ID ordered by `updated_at DESC`.
   - `UpdateCharacterUseCase`: applies partial updates to mutable sheet fields (`name`, `hitPointsCurrent`, `hitPointsTemp`, `notes`).
   - `DeleteCharacterUseCase`: removes a character owned by user ID.
   - `RollAbilityScoresUseCase`: generates or retrieves single-chance `4d6kh3` ability scores.
   - `RollHitPointsUseCase`: generates or retrieves single-chance Hit Die rolls for target level.
2. Define Outbound Ports in `com.dungeoneer.playerCharacter.application.port.out`:
   - `PlayerCharacterRepositoryPort`: `save`, `findByIdAndUserId`, `findAllByUserIdOrderByUpdatedAtDesc`, `deleteByIdAndUserId`, `existsByIdAndUserId`.
   - `AbilityRollRepositoryPort`: `findUnusedByUserId`, `save`, `markAsUsed`.
   - `HpRollRepositoryPort`: `findUnusedByUserIdAndArchetypeIdAndTargetLevel`, `save`, `markAsUsed`.
3. Define Application DTOs and Commands in `com.dungeoneer.playerCharacter.application.dto`:
   - `CreateCharacterCommand`, `UpdateCharacterCommand`, `RollAbilityScoresResult`, `RollHitPointsCommand`, `RollHitPointsResult`, `CharacterSheetResult`, `CharacterSummaryResult`.
4. Implement Application Services in `com.dungeoneer.playerCharacter.application.service`:
   - `CreateCharacterService`: coordinates `AbilityScoreValidator`, `RulesEngine`, roll token consumption (`used = true`), and persistence.
   - `CharacterQueryService`: queries characters strictly scoped by `userId`.
   - `UpdateCharacterService`: loads character by ID + `userId`, applies `updateSheet(...)`, and persists.
   - `DeleteCharacterService`: verifies ownership and removes character.
   - `RollService`: implements single-chance roll logic with idempotency check (returns existing unused roll if present; otherwise rolls new and persists).
5. Write Mockito unit tests in `src/test/java/com/dungeoneer/playerCharacter/application/service/`:
   - TP-01: Character creation flow for standard array, point buy, and rolled scores.
   - TP-02: Idempotent roll generation (returning unused roll on subsequent calls).
   - TE-01: Reused roll token or invalid score permutation rejection.
   - TE-02: Ownership mismatch rejection (throwing `CharacterNotFoundException`).

#### Quality Gates & Verification
- Run application layer unit tests:
  `mvn -f player-character-service/backend/pom.xml test -Dtest="com.dungeoneer.playerCharacter.application.**.*Test"`
- Ready-to-use commit command:
  ```bash
  git add player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/ player-character-service/backend/src/test/java/com/dungeoneer/playerCharacter/application/
  git commit -m "feat(player-character): implement application ports, use cases, services and unit tests"
  ```

#### Phase Handoff & Cumulative Walkthrough
- Append to `openspec/changes/character-crud-backend/walkthrough.md` (pt-BR):
  1. Inbound/outbound ports, commands, and service implementations created.
  2. Application unit test execution results.
  3. Pause and present the commit command to the user before advancing to Phase 4.

---

### Phase 4: Persistence Adapter, JPA Entities & Integration Tests

#### Execution Context
- Scope: Execute Phase 4 only.

#### Implementation Steps
1. Implement JPA Entities in `com.dungeoneer.playerCharacter.adapter.out.persistence`:
   - `ClassProgressionJpaEntity` (`@Table(name = "class_progressions", schema = "player_character")`).
   - `AbilityRollJpaEntity` (`@Table(name = "ability_rolls", schema = "player_character")`) with `int[]` mapping.
   - `HpRollJpaEntity` (`@Table(name = "hp_rolls", schema = "player_character")`) with `int[]` mapping.
   - Update `PlayerCharacterJpaEntity`:
     - Add `notes`, `hp_calculation_method`, `has_pending_level_choices`, enforce `user_id NOT NULL`.
     - Map `@OneToMany` cascade association for `classProgressions`.
2. Define Spring Data JPA Repository Interfaces:
   - `SpringDataPlayerCharacterRepository`: queries filtering by `userId` and sorting by `updatedAt DESC`.
   - `SpringDataAbilityRollRepository`: `findFirstByUserIdAndUsedFalse(UUID userId)`.
   - `SpringDataHpRollRepository`: `findFirstByUserIdAndArchetypeIdAndTargetLevelAndUsedFalse(UUID userId, UUID archetypeId, int targetLevel)`.
   - `SpringDataClassProgressionRepository`.
3. Implement Persistence Adapters in `com.dungeoneer.playerCharacter.adapter.out.persistence`:
   - `PlayerCharacterPersistenceAdapter`: implements outbound repository ports, mapping cleanly between JPA entities and domain models.
4. Write Testcontainers Integration Tests in `src/test/java/com/dungeoneer/playerCharacter/adapter/out/persistence/`:
   - TP-03: Full character persistence lifecycle against PostgreSQL Testcontainer (Flyway V1–V4).
   - TP-02: Idempotent unused roll lookup and atomic `used = true` flag transition.
   - TE-02: User ID isolation: characters belonging to user A cannot be queried or updated by user B.

#### Quality Gates & Verification
- Run persistence integration tests:
  `mvn -f player-character-service/backend/pom.xml test -Dtest="*Persistence*IT,*Persistence*Test"`
- Ready-to-use commit command:
  ```bash
  git add player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/out/persistence/ player-character-service/backend/src/test/java/com/dungeoneer/playerCharacter/adapter/out/persistence/
  git commit -m "feat(player-character): implement persistence adapter, JPA entities, and Testcontainers integration tests"
  ```

#### Phase Handoff & Cumulative Walkthrough
- Append to `openspec/changes/character-crud-backend/walkthrough.md` (pt-BR):
  1. JPA entities and persistence adapters created.
  2. Testcontainers integration test results.
  3. Pause and present the commit command to the user before advancing to Phase 5.

---

### Phase 5: Web Adapter, REST Endpoints, Validation & MockMvc API Tests

#### Execution Context
- Scope: Execute Phase 5 only.

#### Implementation Steps
1. Implement Web DTOs with Jakarta Validation in `com.dungeoneer.playerCharacter.adapter.in.web.dto`:
   - `CreateCharacterRequest`: `@NotBlank @Size(max = 100) name`, `@NotNull lineageId`, `@NotNull backgroundId`, `@NotNull archetypeId`, `@Min(1) @Max(20) level`, `@NotNull abilityScoreMethod`, `baseAbilityScores`, `abilityScoreBonuses`, `abilityRollId`, `hpRollId`.
   - `UpdateCharacterRequest`: `@NotBlank @Size(max = 100) name`, `@Min(0) hitPointsCurrent`, `@Min(0) hitPointsTemp`, `@NotNull String notes`.
   - `RollHpRequest`: `@NotNull UUID archetypeId`, `@Min(2) @Max(20) int level`.
   - Response DTOs: `CharacterSheetResponse` (full hydrated sheet), `CharacterSummaryResponse` (list card view), `AbilityRollResponse`, `HpRollResponse`.
2. Implement REST Controller in `com.dungeoneer.playerCharacter.adapter.in.web.controller`:
   - `PlayerCharacterController`:
     - `POST /api/characters/ability-rolls` (`@RequestHeader("X-User-Id") UUID userId`) -> `201 Created` / `200 OK`.
     - `POST /api/characters/hp-rolls` (`@RequestHeader("X-User-Id") UUID userId`, `@Valid @RequestBody RollHpRequest`) -> `201 Created` / `200 OK`.
     - `POST /api/characters` (`@RequestHeader("X-User-Id") UUID userId`, `@Valid @RequestBody CreateCharacterRequest`) -> `201 Created`.
     - `GET /api/characters` (`@RequestHeader("X-User-Id") UUID userId`) -> `200 OK`.
     - `GET /api/characters/{id}` (`@RequestHeader("X-User-Id") UUID userId`, `@PathVariable UUID id`) -> `200 OK`.
     - `PUT /api/characters/{id}` (`@RequestHeader("X-User-Id") UUID userId`, `@PathVariable UUID id`, `@Valid @RequestBody UpdateCharacterRequest`) -> `200 OK`.
     - `DELETE /api/characters/{id}` (`@RequestHeader("X-User-Id") UUID userId`, `@PathVariable UUID id`) -> `204 No Content`.
3. Enhance `GlobalExceptionHandler` in `com.dungeoneer.core.web`:
   - Map `CharacterNotFoundException` -> `404 Not Found` ProblemDetail.
   - Map `CharacterValidationException` -> `422 Unprocessable Entity` ProblemDetail.
   - Map `RollConflictException` / `InvalidRollTokenException` -> `409 Conflict` / `422 Unprocessable Entity` ProblemDetail.
   - Map `MissingRequestHeaderException` (missing `X-User-Id`) -> `401 Unauthorized` ProblemDetail.
4. Implement MockMvc Web API Tests in `src/test/java/com/dungeoneer/playerCharacter/adapter/in/web/`:
   - TP-01: `POST /api/characters` with `STANDARD_ARRAY`, `POINT_BUY`, and `ROLLED` in 2014 & 2024 rulesets returning `201 Created` with accurate derived stats.
   - TP-02: `POST /api/characters/ability-rolls` returning `201 Created` on initial roll and `200 OK` with identical scores on repeat call.
   - TP-03: Full CRUD lifecycle (`POST` -> `GET` list -> `GET/{id}` -> `PUT/{id}` -> `DELETE/{id}`).
   - TE-01: Reused roll token or tampered score permutation returns `409` / `422`.
   - TE-02: Cross-user access returns `404 Not Found` (IDOR / BOLA prevention).
   - TE-03: Invalid `SRD_2024` background ASI choice returns `422 Unprocessable Entity`.
   - Missing `X-User-Id` header returns `401 Unauthorized`.

#### Quality Gates & Verification
- Run web layer test suite:
  `mvn -f player-character-service/backend/pom.xml test -Dtest="*PlayerCharacterController*Test"`
- Ready-to-use commit command:
  ```bash
  git add player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/in/web/ player-character-service/backend/src/main/java/com/dungeoneer/core/web/ player-character-service/backend/src/test/java/com/dungeoneer/playerCharacter/adapter/in/web/
  git commit -m "feat(player-character): implement character CRUD REST endpoints, validation, and web tests"
  ```

#### Phase Handoff & Cumulative Walkthrough
- Append to `openspec/changes/character-crud-backend/walkthrough.md` (pt-BR):
  1. Web controller, DTOs, and exception handler updates.
  2. MockMvc API test execution results.
  3. Pause and present the commit command to the user before advancing to Phase 6.

---

### Phase 6: Full Verification Gate, Spec Deltas & Final Walkthrough Consolidation

#### Execution Context
- Scope: Execute Phase 6. Run full build verification, verify JaCoCo coverage gate, update specification deltas, and prepare change for archival.

#### Implementation Steps
1. Run full Maven verification and coverage gate:
   - Ensure JaCoCo line coverage is >= 80% on domain and application layers:
     `mvn -f player-character-service/backend/pom.xml clean verify`
2. Update Specification Deltas:
   - `openspec/changes/character-crud-backend/specs/player-character-domain-delta.md`: document endpoints, roll generation & single-use consumption, ruleset ASI rules, HP calculations, sheet update constraints, and `X-User-Id` ownership.
   - `openspec/changes/character-crud-backend/specs/player-character-data-model-delta.md`: document `ability_rolls`, `hp_rolls`, `class_progressions` tables, `notes`, `hp_calculation_method`, `has_pending_level_choices`, and `user_id NOT NULL` constraint on `characters`.
3. Update `openspec/changes/character-crud-backend/tasks.md`:
   - Mark all completed tasks with `[x]`.
4. Finalize `openspec/changes/character-crud-backend/walkthrough.md` in Portuguese (pt-BR):
   - Consolidate all changes executed across Phases 1 through 6.
   - Document full test verification and coverage summary metrics.
   - Provide clear actionable next steps (frontend wizard integration, `sdd-spec-validate`, or archival via `sdd-spec-archive`).

#### Quality Gates & Verification
- Run full Maven build:
  `mvn -f player-character-service/backend/pom.xml clean verify`
- Ready-to-use commit command:
  ```bash
  git add openspec/changes/character-crud-backend/ prompt-character-crud-backend.md
  git commit -m "chore(player-character): complete verification, delta spec documentation, and walkthrough"
  ```

#### Final Phase Handoff
- Present the consolidated walkthrough to the user and notify readiness for `sdd-spec-validate` or `sdd-spec-archive`.
