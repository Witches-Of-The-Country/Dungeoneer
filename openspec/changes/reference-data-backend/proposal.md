# Proposal: Reference Data Backend

> **Change ID:** `reference-data-backend`
> **Status:** draft
> **Author:** @jaca
> **Created:** 2026-10-29
> **Last Updated:** 2026-10-29

---

## Summary

Establish the isolated `player_character` PostgreSQL schema in `player-character-service`, migrate existing tables (`characters`, `ability_scores`) into `player_character`, create the core reference data tables (`archetypes`, `lineages`, `backgrounds`), seed all 13 SRD classes, 9 lineages, and 6 backgrounds across both `SRD_2014` and `SRD_2024` rulesets via Flyway migrations, and expose read-only reference endpoints (`GET /api/reference/classes`, `GET /api/reference/lineages`, `GET /api/reference/backgrounds`) with optional `?ruleset=` filtering.

## Motivation

The Character Creation Wizard requires authoritative reference data for Lineages, Classes (`Archetypes`), and Backgrounds in both `SRD_2014` and `SRD_2024` rulesets—including hit dice, saving throw/skill proficiencies, 2014 fixed lineage Ability Score Increases (ASI), 2024 background allowed ASI abilities + origin feat summary text, and speed/size. Currently, `player-character-service` has no reference tables, no seed data, and its existing tables reside in the `public` schema instead of `player_character`.

## Scope

### In Scope
- Flyway migration `V2__align_schema_and_create_reference_tables.sql`:
  - Create schema `player_character` (`CREATE SCHEMA IF NOT EXISTS player_character;`) and move `public.characters` and `public.ability_scores` into `player_character`.
  - Update `PlayerCharacterJpaEntity` and `AbilityScoresJpaEntity` `@Table` annotations with `schema = "player_character"`.
  - Create tables `player_character.archetypes`, `player_character.lineages`, and `player_character.backgrounds`:
    - `archetypes`: `id UUID PK`, `name VARCHAR(50) NOT NULL`, `hit_dice VARCHAR(3) NOT NULL` (`D6`, `D8`, `D10`, `D12`), `primary_ability VARCHAR(15) NOT NULL`, `saving_throw_proficiencies VARCHAR[] NOT NULL`, `armor_proficiencies VARCHAR[]`, `weapon_proficiencies VARCHAR[]`, `spellcasting_type VARCHAR(15) NOT NULL` (`FULL`, `HALF`, `THIRD`, `PACT`, `NONE`), `spellcasting_ability VARCHAR(15)`, `ruleset VARCHAR(10) NOT NULL`, `source VARCHAR(10) NOT NULL DEFAULT 'SRD'`, `owner_id UUID NULL`, `UNIQUE (name, ruleset, source)`.
    - `lineages`: `id UUID PK`, `name VARCHAR(50) NOT NULL`, `description TEXT`, `speed INT NOT NULL DEFAULT 30`, `size VARCHAR(10) NOT NULL`, `ability_bonuses JSONB NOT NULL DEFAULT '{}'` (for `SRD_2014` fixed ASI, e.g. `{"DEX": 2}` or `{"ALL": 1}`, empty `{}` in `SRD_2024`), `traits_summary TEXT[]`, `ruleset VARCHAR(10) NOT NULL`, `source VARCHAR(10) NOT NULL DEFAULT 'SRD'`, `owner_id UUID NULL`, `UNIQUE (name, ruleset, source)`.
    - `backgrounds`: `id UUID PK`, `name VARCHAR(50) NOT NULL`, `description TEXT`, `skill_proficiencies VARCHAR[] NOT NULL`, `tool_proficiencies VARCHAR[]`, `languages VARCHAR[]`, `allowed_ability_bonuses VARCHAR[]` (for `SRD_2024` 3 allowed abilities for `+2/+1` or `+1/+1/+1`, empty in `SRD_2014`), `origin_feat_name VARCHAR(100) NULL` (informational text for `SRD_2024`), `origin_feat_description TEXT NULL`, `ruleset VARCHAR(10) NOT NULL`, `source VARCHAR(10) NOT NULL DEFAULT 'SRD'`, `owner_id UUID NULL`, `UNIQUE (name, ruleset, source)`.
  - Add Foreign Keys on `player_character.characters` (`lineage_id -> lineages.id`, `background_id -> backgrounds.id`).
- Flyway migration `V3__seed_srd_2014_and_2024_reference_data.sql`:
  - Seed all **13 Classes** (`Barbarian`, `Bard`, `Cleric`, `Druid`, `Fighter`, `Monk`, `Paladin`, `Ranger`, `Rogue`, `Sorcerer`, `Warlock`, `Wizard`, `Artificer`) for both `SRD_2014` and `SRD_2024`.
  - Seed all **9 Lineages** (`Human`, `Elf`, `Dwarf`, `Halfling`, `Gnome`, `Half-Elf`, `Half-Orc`, `Tiefling`, `Dragonborn`) for both `SRD_2014` (with fixed `ability_bonuses`) and `SRD_2024` (cosmetic/traits, empty `ability_bonuses`).
  - Seed all **6 Backgrounds** (`Acolyte`, `Criminal`, `Folk Hero`, `Noble`, `Sage`, `Soldier`) for both `SRD_2014` (skills/tools/languages) and `SRD_2024` (skills + 3 allowed ASI abilities + informational Origin Feat name & description).
- Implement pure Java domain models (`Archetype`, `Lineage`, `Background`, `HitDiceEnum`, `SpellcastingType`), outbound repository ports (`ReferenceDataRepository`), JPA entities & adapters, and `GetReferenceDataUseCase`.
- Implement `ReferenceDataController`:
  - `GET /api/reference/classes?ruleset=SRD_2014|SRD_2024`
  - `GET /api/reference/lineages?ruleset=SRD_2014|SRD_2024`
  - `GET /api/reference/backgrounds?ruleset=SRD_2014|SRD_2024`

### Out of Scope
- Subclasses, spells (`~300`), feats (`~40`), items, and homebrew creation (`POST /api/reference/homebrew/*`).

## Proposed Solution

### Approach
Following Hexagonal Architecture, `Archetype`, `Lineage`, and `Background` in `domain/model/` will be implemented as immutable, framework-free domain classes. Outbound persistence adapters will map PostgreSQL array/JSONB columns to domain collections. `ReferenceDataController` will expose the 3 GET endpoints with optional `ruleset` query parameter validation (`SRD_2014` or `SRD_2024`), returning `422 Unprocessable Entity` ProblemDetail if an invalid ruleset filter is supplied.

### Impacted Files
| File | Change Type | Notes |
|------|-------------|-------|
| `player-character-service/backend/src/main/resources/db/migration/V2__align_schema_and_create_reference_tables.sql` | Create | Create `player_character` schema, move existing tables, create `archetypes`, `lineages`, `backgrounds` |
| `player-character-service/backend/src/main/resources/db/migration/V3__seed_srd_2014_and_2024_reference_data.sql` | Create | Seed 13 classes, 9 lineages, 6 backgrounds for `SRD_2014` and `SRD_2024` |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/domain/model/Archetype.java` | Modify | Replace empty stub with rich domain entity |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/domain/model/Lineage.java` | Modify | Replace empty stub with rich domain entity (including 2014 ASI bonuses map) |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/domain/model/Background.java` | Modify | Replace empty stub with rich domain entity (including 2024 allowed ASI abilities & origin feat info) |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/domain/model/HitDiceEnum.java` | Create | Enum `D6(6)`, `D8(8)`, `D10(10)`, `D12(12)` with `averagePerLevel()` helper |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/port/in/GetReferenceDataUseCase.java` | Create | Inbound port for listing classes, lineages, and backgrounds |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/port/out/ReferenceDataRepository.java` | Create | Outbound port for querying reference data by ID and ruleset |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/service/ReferenceDataService.java` | Create | Application service implementing `GetReferenceDataUseCase` |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/out/persistence/PlayerCharacterJpaEntity.java` | Modify | Add `schema = "player_character"` to `@Table` |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/out/persistence/AbilityScoresJpaEntity.java` | Modify | Add `schema = "player_character"` to `@Table` |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/out/persistence/ArchetypeJpaEntity.java` | Create | JPA entity for `player_character.archetypes` |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/out/persistence/LineageJpaEntity.java` | Create | JPA entity for `player_character.lineages` |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/out/persistence/BackgroundJpaEntity.java` | Create | JPA entity for `player_character.backgrounds` |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/out/persistence/ReferenceDataPersistenceAdapter.java` | Create | Persistence adapter implementing `ReferenceDataRepository` |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/in/web/controller/ReferenceDataController.java` | Create | REST controller for `/api/reference/classes`, `/lineages`, `/backgrounds` |

### Data Model Changes (if applicable)
Moves `characters` and `ability_scores` to schema `player_character` and creates `player_character.archetypes`, `player_character.lineages`, and `player_character.backgrounds` with dual-ruleset seed data.

## Acceptance Criteria

- [ ] **Given** a fresh PostgreSQL database **When** Flyway migrations `V1`–`V3` execute **Then** schema `player_character` contains `characters`, `ability_scores`, `archetypes` (26 rows: 13 per ruleset), `lineages` (18 rows: 9 per ruleset), and `backgrounds` (12 rows: 6 per ruleset).
- [ ] **Given** `GET /api/reference/classes?ruleset=SRD_2024` **When** requested **Then** `200 OK` returns all 13 `SRD_2024` classes with hit dice, primary ability, proficiencies, and spellcasting metadata.
- [ ] **Given** `GET /api/reference/lineages?ruleset=SRD_2014` **When** requested **Then** `200 OK` returns all 9 `SRD_2014` lineages including their fixed `abilityBonuses`.
- [ ] **Given** `GET /api/reference/backgrounds?ruleset=SRD_2024` **When** requested **Then** `200 OK` returns all 6 `SRD_2024` backgrounds including `allowedAbilityBonuses` (3 abilities) and informational `originFeatName` / `originFeatDescription`.

## Defensive Security & Validation Considerations

### Input Validation
- Query parameter `ruleset` is validated against the `Ruleset` enum (`SRD_2014`, `SRD_2024`); invalid values return `422 Unprocessable Entity` ProblemDetail.

### Secret Isolation
- Database credentials remain injected exclusively via environment variables (`${DB_PASSWORD}`).

### Authorization & Authentication Impact
- `/api/reference/**` endpoints expose public SRD reference data (`GET` only) and do not mutate state or expose user data.

### Other Security Risks
- None; all queries use parameterized Spring Data JPA methods.

## Testing Strategy & Shift-Left Plan

### Happy Path Tests
| Test ID | Scenario | Test Type | Expected Result |
|---------|----------|-----------|-----------------|
| TP-01 | Domain construction and invariants of `Archetype`, `Lineage`, `Background` | Unit (JUnit 5) | Valid instances created with expected getters and ASI metadata |
| TP-02 | Flyway `V2` and `V3` migrations and `ReferenceDataPersistenceAdapter` queries by ruleset | Integration (Testcontainers) | Returns 13 classes, 9 lineages, and 6 backgrounds per ruleset |
| TP-03 | `GET /api/reference/{classes,lineages,backgrounds}` with and without `ruleset` filter | API (MockMvc) | Returns `200 OK` with expected JSON array |

### Edge Case & Failure Mode Tests
| Test ID | Scenario | Test Type | Expected Result |
|---------|----------|-----------|-----------------|
| TE-01 | `GET /api/reference/classes?ruleset=INVALID` | API (MockMvc) | Returns `422 Unprocessable Entity` RFC 7807 `ProblemDetail` |
| TE-02 | Domain model rejects blank names or null ruleset | Unit (JUnit 5) | Throws `CharacterValidationException` / `IllegalArgumentException` |

### Test Commands
- **Run tests:** `mvn -f player-character-service/backend/pom.xml clean test`
- **Coverage check:** `mvn -f player-character-service/backend/pom.xml clean verify`

## Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Moving `characters` to `player_character` schema breaks existing `PlayerCharacterPersistenceAdapterTest` | Medium | High | Update `@Table(schema = "player_character")` on existing JPA entities and verify all existing tests in the same commit |

## Specification Changes & Delta Stubs

### `specs/player-character-data-model-delta.md`
- **Target Canonical Spec:** `openspec/specs/player-character-data-model.md`
- **Summary of Updates:** Document `ability_bonuses` (`JSONB`) and `traits_summary` on `lineages`, `allowed_ability_bonuses` and `origin_feat_name`/`origin_feat_description` on `backgrounds`, and clarify that `characters.user_id` is a `NOT NULL UUID` logical reference to `auth-service` without a cross-schema SQL foreign key.

## References

- `openspec/specs/player-character-data-model.md`
- `openspec/specs/player-character-domain.md`
