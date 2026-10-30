# Proposal: Character Crud Backend

> **Change ID:** `character-crud-backend`
> **Status:** draft
> **Author:** @jaca
> **Created:** 2026-10-29
> **Last Updated:** 2026-10-29

---

## Summary

Implement complete, user-scoped Player Character CRUD (`POST /api/characters`, `GET /api/characters`, `GET /api/characters/{id}`, `PUT /api/characters/{id}`, `DELETE /api/characters/{id}`) and single-chance server-side roll endpoints (`POST /api/characters/ability-rolls` for `4d6kh3` and `POST /api/characters/hp-rolls` for rolled HP above level 1) in `player-character-service`. The backend acts as the single source of truth for ability score validation (`STANDARD_ARRAY`, `POINT_BUY` 27-point budget, `ROLLED` single-use DB verification), ruleset-specific Ability Score Increases (`SRD_2014` lineage ASI vs. `SRD_2024` background `+2/+1` or `+1/+1/+1` ASI), initial level (1–20) HP calculation (`MAX`, `AVERAGE`, `ROLLED`), derived stats (`AC`, `initiative`, `proficiencyBonus`, `speed`), initial `ClassProgression` persistence, `notes` field persistence, and strict ownership enforcement via the `X-User-Id` header.

## Motivation

Currently, `player-character-service` only has a partial `POST /api/characters` endpoint with hardcoded `hitPointsMax = 1` and `AbilityScoreMethod.POINT_BUY` in `PlayerCharacterWebMapper`, no `GET`/`PUT`/`DELETE` endpoints, no server-side validation for Point Buy / Standard Array / `4d6kh3` single-chance rolls, no ASI application, no `class_progressions` table, no `notes` column, and no `user_id` ownership enforcement.

## Scope

### In Scope
- **Database Migration (`V4__add_character_crud_and_roll_tables.sql`)**:
  - Alter `player_character.characters`: add `notes TEXT NOT NULL DEFAULT ''`, `hp_calculation_method VARCHAR(15) NOT NULL DEFAULT 'MAX'`, `has_pending_level_choices BOOLEAN NOT NULL DEFAULT FALSE`, and enforce `user_id UUID NOT NULL`.
  - Create `player_character.class_progressions` (`id UUID PK`, `character_id UUID FK`, `archetype_id UUID FK`, `subclass_id UUID NULL`, `class_level INT NOT NULL CHECK (1-20)`, `UNIQUE (character_id, archetype_id)`).
  - Create `player_character.ability_rolls` (`id UUID PK`, `user_id UUID NOT NULL`, `rolled_scores INT[] NOT NULL` — array of 6 integers 3–18 generated via `4d6kh3`, `used BOOLEAN NOT NULL DEFAULT FALSE`, `created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()`).
  - Create `player_character.hp_rolls` (`id UUID PK`, `user_id UUID NOT NULL`, `archetype_id UUID NOT NULL FK`, `target_level INT NOT NULL CHECK (2-20)`, `rolled_hp_per_level INT[] NOT NULL`, `used BOOLEAN NOT NULL DEFAULT FALSE`, `created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()`).
- **Single-Chance Server-Side Roll Endpoints**:
  - `POST /api/characters/ability-rolls` (requires `X-User-Id`): If the user already has an unused `ability_rolls` record, return that existing record (`200 OK`) so refreshing the wizard does not grant a new roll; otherwise roll 6× `4d6kh3` (including individual 4d6 dice breakdown for UI display), persist with `used = false`, and return `201 Created` with `rollId` and the 6 values.
  - `POST /api/characters/hp-rolls` (requires `X-User-Id`, body `{ archetypeId, level }`): If the user already has an unused `hp_rolls` record for the same `archetypeId` and `target_level`, return it; otherwise roll the class Hit Die for levels `2..level`, persist with `used = false`, and return `201 Created`.
- **Server-Side Rules & Creation (`POST /api/characters`)**:
  - Require `X-User-Id` header (`UUID`).
  - Validate base ability scores according to `abilityScoreMethod`:
    - `STANDARD_ARRAY`: base scores must be an exact permutation of `[15, 14, 13, 12, 10, 8]`.
    - `POINT_BUY`: each base score in `8..15` and total cost (`8:0, 9:1, 10:2, 11:3, 12:4, 13:5, 14:7, 15:9`) must be `<= 27` (exact 27 required on creation).
    - `ROLLED`: require `abilityRollId`, verify it belongs to `X-User-Id` and `used == false`, verify the 6 assigned base scores are an exact multiset permutation of `rolled_scores`, and mark `used = true` atomically.
  - Apply ruleset-specific ASI via `RulesEngine`:
    - `SRD_2014`: apply fixed `ability_bonuses` from the selected `Lineage` (plus choice if Half-Elf `+1/+1`).
    - `SRD_2024`: validate that the player's `abilityScoreBonuses` map (`+2/+1` across 2 distinct abilities or `+1/+1/+1` across 3 distinct abilities) uses only abilities listed in the selected `Background.allowedAbilityBonuses`, and add them to base scores (capped at 20).
  - Calculate `hitPointsMax` on the server:
    - Level 1 always receives `hitDice.max + CON_mod` (minimum 1).
    - For `level > 1`:
      - `MAX`: each level `2..level` adds `hitDice.max + CON_mod`.
      - `AVERAGE`: each level `2..level` adds `(hitDice.max / 2 + 1) + CON_mod`.
      - `ROLLED`: require `hpRollId`, verify ownership, `used == false`, `archetypeId`, and `targetLevel == level`, sum `rolled_hp_per_level + CON_mod` per level, and mark `used = true`.
    - Set `hasPendingLevelChoices = (level > 1)`.
  - Create initial `ClassProgression(archetypeId, level)` and persist `PlayerCharacter` + `AbilityScores` + `ClassProgression`.
- **Read, Update & Delete Endpoints**:
  - `GET /api/characters`: Return all characters owned by `X-User-Id` ordered by `updated_at DESC`, enriched with lineage, background, and class progression names/details.
  - `GET /api/characters/{id}`: Return full character sheet DTO (including hydrated `Lineage`, `Background` with skill proficiencies & origin feat text, `ClassProgression` with `Archetype` saving throws & hit dice, derived 18 skills modifiers, and `notes`). Return `404 Not Found` if not found or owned by another user.
  - `PUT /api/characters/{id}`: Allow updating **only** `name` (1–100 chars), `hitPointsCurrent` (`0 <= hp <= hitPointsMax`), `hitPointsTemp` (`>= 0`), and `notes`. Level, class, lineage, background, and ability scores are immutable via `PUT`.
  - `DELETE /api/characters/{id}`: Physical deletion of character (cascading to `ability_scores` and `class_progressions`) if owned by `X-User-Id`, returning `204 No Content`.
- **Cleanup & Error Handling**:
  - Delete duplicate dormant stubs (`CreatePlayerCharacterUseCase`, `CharacterRepositoryPort`, `CreatePlayerCharacterService`, `PlayerCharacterUseCaseConfig`).
  - Expand `GlobalExceptionHandler` to map `CharacterNotFoundException` (`404`), `CharacterValidationException` (`422`), `InvalidRollTokenException` / `RollAlreadyUsedException` (`409` / `422`), and missing `X-User-Id` (`401`).

### Out of Scope
- Level-up (`POST /api/characters/{id}/level-up`) and Multiclass (`POST /api/characters/{id}/multiclass`) interactive choice wizard (deferred to future spec).

## Proposed Solution

### Approach
We introduce a pure domain `RulesEngine` strategy (`Srd2014RulesEngine`, `Srd2024RulesEngine`) and `AbilityScoreValidator` inside `domain/model/` (zero framework imports) to validate base scores, apply ASI, and compute HP/AC/Initiative/Proficiency/Speed. `PlayerCharacterController` extracts `@RequestHeader("X-User-Id") UUID userId` on every endpoint and delegates to inbound use cases (`CreateCharacterUseCase`, `GetCharacterUseCase`, `ListCharactersUseCase`, `UpdateCharacterUseCase`, `DeleteCharacterUseCase`, `RollAbilityScoresUseCase`, `RollHitPointsUseCase`).

### Impacted Files
| File | Change Type | Notes |
|------|-------------|-------|
| `player-character-service/backend/src/main/resources/db/migration/V4__add_character_crud_and_roll_tables.sql` | Create | Add `notes`, `class_progressions`, `ability_rolls`, `hp_rolls`, enforce `user_id NOT NULL` |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/domain/model/PlayerCharacter.java` | Modify | Support `notes`, `hpCalculationMethod`, `hasPendingLevelChoices`, `ClassProgression`, and `updateSheet()` |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/domain/model/ClassProgression.java` | Modify | Implement domain entity for class & level progression |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/domain/model/RulesEngine.java` | Create | Strategy for `SRD_2014` vs `SRD_2024` ASI and character creation rules |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/domain/model/AbilityScoreValidator.java` | Create | Validates `STANDARD_ARRAY`, `POINT_BUY` (27 pts), and `ROLLED` multiset permutation |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/port/in/` | Modify | Add `GetCharacterUseCase`, `ListCharactersUseCase`, `UpdateCharacterUseCase`, `DeleteCharacterUseCase`, `RollAbilityScoresUseCase`, `RollHitPointsUseCase`; delete duplicate stubs |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/service/` | Modify | Implement CRUD and roll services; delete `CreatePlayerCharacterService` stub |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/out/persistence/` | Modify | Map `class_progressions`, `ability_rolls`, `hp_rolls`, `notes`, and user-scoped queries |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/in/web/controller/PlayerCharacterController.java` | Modify | Add `POST /ability-rolls`, `POST /hp-rolls`, `GET`, `GET/{id}`, `PUT/{id}`, `DELETE/{id}` with `X-User-Id` |
| `player-character-service/backend/src/main/java/com/dungeoneer/core/web/GlobalExceptionHandler.java` | Modify | Map `CharacterNotFoundException` (`404`), `CharacterValidationException` (`422`), `RollConflictException` (`409`), missing `X-User-Id` (`401`) |

### Data Model Changes (if applicable)
Adds `notes`, `hp_calculation_method`, and `has_pending_level_choices` to `player_character.characters`, and creates `player_character.class_progressions`, `player_character.ability_rolls`, and `player_character.hp_rolls`.

## Acceptance Criteria

- [ ] **Given** an authenticated user (`X-User-Id`) **When** `POST /api/characters/ability-rolls` is called twice **Then** the first call rolls and persists 6 `4d6kh3` values (`201 Created`) and the second call returns the exact same unused roll (`200 OK`) without generating a new roll.
- [ ] **Given** an unused `abilityRollId` **When** `POST /api/characters` is called with `abilityScoreMethod = ROLLED` and base scores that match a permutation of the rolled values **Then** the roll is marked `used = true`, ruleset ASI (`2014` lineage or `2024` background) is applied, HP/AC/Initiative/Proficiency are calculated server-side, and `201 Created` returns the full character sheet DTO.
- [ ] **Given** `POST /api/characters` with `POINT_BUY` exceeding 27 points or `ROLLED` with tampered numbers **When** called **Then** `422 Unprocessable Entity` ProblemDetail is returned.
- [ ] **Given** `GET /api/characters` with `X-User-Id` **When** called **Then** only characters belonging to that `user_id` are returned, ordered by `updated_at DESC`.
- [ ] **Given** `PUT /api/characters/{id}` with updated `name`, `hitPointsCurrent`, `hitPointsTemp`, and `notes` **When** called by the owner **Then** `200 OK` returns the updated sheet; if called by a different `X-User-Id`, `404 Not Found` is returned.
- [ ] **Given** `DELETE /api/characters/{id}` **When** called by the owner **Then** the character is deleted and `204 No Content` is returned.

## Defensive Security & Validation Considerations

### Input Validation
- All request DTOs validated with Jakarta Validation (`@NotBlank`, `@Size(max = 100)`, `@Min(1)`, `@Max(20)`, `@NotNull`).
- Ability score permutation and Point Buy budget validated strictly in the domain layer; client-supplied derived values are ignored.

### Secret Isolation
- No secrets introduced; DB credentials remain injected via `${DB_PASSWORD}`.

### Authorization & Authentication Impact
- Every endpoint on `/api/characters/**` requires `X-User-Id` and scopes all DB queries by `id AND user_id` to prevent Insecure Direct Object Reference (IDOR / Broken Object Level Authorization).

### Other Security Risks
- Returning `404 Not Found` when a character ID belongs to another user prevents UUID enumeration across tenants.

## Testing Strategy & Shift-Left Plan

### Happy Path Tests
| Test ID | Scenario | Test Type | Expected Result |
|---------|----------|-----------|-----------------|
| TP-01 | Create character with `STANDARD_ARRAY`, `POINT_BUY` (27 pts), and `ROLLED` (valid `abilityRollId`) in both `SRD_2014` and `SRD_2024` | Unit & API (MockMvc) | `201 Created` with accurate ASI, HP (`MAX`/`AVERAGE`/`ROLLED`), AC, and proficiency bonus |
| TP-02 | Single-chance `POST /api/characters/ability-rolls` idempotency when unused roll exists | Unit & Integration | Returns existing unused roll without re-rolling |
| TP-03 | Full CRUD lifecycle (`POST`, `GET` list, `GET/{id}`, `PUT/{id}`, `DELETE/{id}`) | Integration (Testcontainers) & API (MockMvc) | Persists, updates allowed fields, and deletes cleanly |

### Edge Case & Failure Mode Tests
| Test ID | Scenario | Test Type | Expected Result |
|---------|----------|-----------|-----------------|
| TE-01 | Attempt to reuse an already consumed `abilityRollId` or tampered score permutation | Unit & API (MockMvc) | Returns `409 Conflict` / `422 Unprocessable Entity` ProblemDetail |
| TE-02 | Cross-user access (`GET/{id}`, `PUT/{id}`, `DELETE/{id}` with different `X-User-Id`) | Unit & API (MockMvc) | Returns `404 Not Found` ProblemDetail (IDOR protection) |
| TE-03 | Invalid `SRD_2024` background ASI choice (ability not in background's allowed list) | Unit | Throws `CharacterValidationException` → `422` |

### Test Commands
- **Run tests:** `mvn -f player-character-service/backend/pom.xml clean test`
- **Coverage check:** `mvn -f player-character-service/backend/pom.xml clean verify`

## Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Race condition on single-chance roll creation | Low | Medium | Enforce partial unique index `CREATE UNIQUE INDEX idx_unused_ability_roll ON player_character.ability_rolls(user_id) WHERE used = FALSE;` |

## Specification Changes & Delta Stubs

### `specs/player-character-domain-delta.md`
- **Target Canonical Spec:** `openspec/specs/player-character-domain.md`
- **Summary of Updates:** Add `POST /api/characters/ability-rolls` and `POST /api/characters/hp-rolls`, document server-side ASI & HP calculation (`MAX`, `AVERAGE`, `ROLLED`) for levels 1–20, document `PUT /api/characters/{id}` mutable fields (`name`, `hitPointsCurrent`, `hitPointsTemp`, `notes`), and document `X-User-Id` ownership enforcement.

### `specs/player-character-data-model-delta.md`
- **Target Canonical Spec:** `openspec/specs/player-character-data-model.md`
- **Summary of Updates:** Document `ability_rolls` and `hp_rolls` tables, `notes`, `hp_calculation_method`, and `has_pending_level_choices` columns on `characters`, and `user_id NOT NULL`.

## References

- `openspec/specs/player-character-domain.md`
- `openspec/specs/player-character-data-model.md`
