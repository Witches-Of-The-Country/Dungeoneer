# Change Proposal: 001 — Fix HTTP 500 Errors on Character & Item Creation and Enhance RFC 7807 Diagnostics

> **Change ID:** `001-fix-500-errors-and-rfc7807-diagnostics`
> **Status:** approved
> **Created:** 2026-11-01

---

## Summary

Fix the two critical HTTP 500 Internal Server Errors occurring on `POST /api/characters` (character creation with Level > 1 / Origin Feat) and `POST /api/reference/items` (Homebrew item creation with mechanical properties), and upgrade backend `GlobalExceptionHandler` and frontend `apiFetch` to always return and display actionable RFC 7807 `ProblemDetail` diagnostic messages instead of opaque 500 payloads.

---

## Motivation & Problem Statement

1. **`POST /api/characters` HTTP 500**:
   - In `CreateCharacterService` (and `LevelUpService`), when instantiating `CharacterFeat`, the 7th constructor parameter `String choicesJson` (mapped to `character_feats.choices JSONB NOT NULL DEFAULT '{}'::jsonb`) was passed the feat type string (`"ORIGIN"`, `"CLASS"`, `"SUBCLASS"`) instead of valid JSON (`"{}"`). PostgreSQL rejects the unquoted token with `PSQLException: ERROR: invalid input syntax for type json`.
   - Furthermore, `CreateCharacterService` saved `PlayerCharacter` (which cascades `characterFeats`) and then looped over `initialCharacterFeats` calling `saveCharacterFeat(cf)` a second time, violating the `uq_character_feat (character_id, feat_id)` unique constraint.
2. **`POST /api/reference/items` HTTP 500**:
   - `CreateHomebrewItemRequest` declared `ItemMechanics mechanicalProperties` directly as the pure-domain sealed interface `ItemMechanics`. Because the domain layer has zero Jackson annotations (`@JsonTypeInfo`), Jackson fails with `InvalidDefinitionException` when deserializing incoming JSON objects (`{ damageRolls: [...] }`) from the frontend.
   - Additionally, `ItemJpaEntity` mapped `mechanical_properties` (`JSONB`) as a plain `String` without ensuring the `"type"` discriminator (`"WEAPON"`, `"ARMOR"`, `"CONSUMABLE"`) is written during serialization, breaking round-trip deserialization.
3. **Opaque Error Messages**:
   - `GlobalExceptionHandler` in `player-character-service` lacked handlers for `DataIntegrityViolationException` and a fallback `@ExceptionHandler(Exception.class)`, causing unhandled exceptions to fall through to Spring Boot's default error response (`{"timestamp":...,"status":500,"error":"Internal Server Error","path":...}`) without `detail`.

---

## Scope

### In Scope
- Fixing `CharacterFeat` `choicesJson` initialization (`"{}"` or valid JSON object) and deduplicating initial feats in `CreateCharacterService` and `LevelUpService`.
- Removing the redundant `saveCharacterFeat` loop after `playerCharacterRepository.save(character)` in `CreateCharacterService`.
- Adding defensive JSON sanitization in `CharacterFeatJpaEntity.fromDomain()`.
- Replacing raw sealed interface `ItemMechanics` in `CreateHomebrewItemRequest` with a structured boundary DTO (`HomebrewItemMechanicsRequest`) and mapping it in `ReferenceDataWebMapper`.
- Ensuring `ItemJpaEntity.serializeMechanics()` writes `"type": "WEAPON" | "ARMOR" | "CONSUMABLE"` so `deserializeMechanics()` round-trips losslessly.
- Adding `@ExceptionHandler(DataIntegrityViolationException.class)` and fallback `@ExceptionHandler(Exception.class)` in `GlobalExceptionHandler` returning RFC 7807 `ProblemDetail`.
- Updating `dungeoneer-frontend/lib/api/client.ts` to extract and format RFC 7807 `detail`, `title`, and fallback messages clearly in pt-BR.

### Out of Scope
- Adding new multi-roll weapon mechanics or prerequisite builders (covered in `002-advanced-homebrew-mechanics-and-prerequisites`).
- Seeding additional SRD classes/subclasses (covered in `003-complete-srd-classes-subclasses-and-feat-filters`).

---

## Proposed Solution

1. **Fix `CharacterFeat` JSONB & Cascade Persistence (`CreateCharacterService`, `LevelUpService`, `CharacterFeatJpaEntity`)**:
   - Pass `"{\"source\":\"" + sourceType + "\"}"` or `"{}"` as `choicesJson` when creating `CharacterFeat` instances in `CreateCharacterService` and `LevelUpService`.
   - Deduplicate `initialCharacterFeats` by `featId` and remove the redundant `for (CharacterFeat cf : initialCharacterFeats) { playerCharacterRepository.saveCharacterFeat(cf); }` loop after `playerCharacterRepository.save(character)`.
   - Add defensive JSON normalization in `CharacterFeatJpaEntity.fromDomain()` so any non-JSON string is wrapped as `{"source": "..."}` before binding to PostgreSQL `JSONB`.
2. **Fix Homebrew Item Mechanics Deserialization & JSONB Persistence (`CreateHomebrewItemRequest`, `ReferenceDataWebMapper`, `ItemJpaEntity`)**:
   - Introduce a structured `HomebrewItemMechanicsRequest` DTO in `application/dto/` that Jackson deserializes cleanly, and map it in `ReferenceDataWebMapper` to `WeaponMechanics`, `ArmorMechanics`, `ConsumableMechanics`, or `EmptyItemMechanics` based on `itemCategory` or explicit fields, preserving Clean Architecture in `domain/model`.
   - Update `ItemJpaEntity.serializeMechanics()` to always include `"type": "WEAPON" | "ARMOR" | "CONSUMABLE"` in the serialized JSONB object so `deserializeMechanics()` round-trips losslessly.
3. **Enhance RFC 7807 `ProblemDetail` Error Diagnostics (`GlobalExceptionHandler`, `client.ts`)**:
   - Add `@ExceptionHandler(DataIntegrityViolationException.class)` (HTTP 409 Conflict with clear message in pt-BR) and `@ExceptionHandler(Exception.class)` (HTTP 500 `ProblemDetail` with safe `title`, `detail`, `status`, and `path`) in `GlobalExceptionHandler`.
   - Update `dungeoneer-frontend/lib/api/client.ts` to extract `detail`, `message`, `error`, and `path` so the user sees clear toast messages in pt-BR.

---

## Impacted Files

| File Path | Layer | Reason |
|---|---|---|
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/service/CreateCharacterService.java` | Application | Fix `CharacterFeat` `choicesJson`, deduplicate by `featId`, and remove duplicate `saveCharacterFeat` loop. |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/service/LevelUpService.java` | Application | Fix `CharacterFeat` `choicesJson` valid JSON format. |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/out/persistence/CharacterFeatJpaEntity.java` | Persistence | Ensure `choices` is always valid JSON before persisting to JSONB. |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/dto/CreateHomebrewItemRequest.java` | Application DTO | Accept structured `HomebrewItemMechanicsRequest` DTO for `mechanicalProperties`. |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/in/web/mapper/ReferenceDataWebMapper.java` | Web Mapper | Map `CreateHomebrewItemRequest.mechanicalProperties` to `WeaponMechanics`, `ArmorMechanics`, or `ConsumableMechanics`. |
| `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/adapter/out/persistence/ItemJpaEntity.java` | Persistence | Include `"type"` discriminator when serializing `ItemMechanics` to JSONB. |
| `player-character-service/backend/src/main/java/com/dungeoneer/core/web/GlobalExceptionHandler.java` | Web Infrastructure | Add `DataIntegrityViolationException` and fallback `Exception` handlers returning RFC 7807 `ProblemDetail`. |
| `dungeoneer-frontend/lib/api/client.ts` | Frontend API | Improve RFC 7807 and fallback error message formatting in `apiFetch`. |

---

## Acceptance Criteria

- **Given** a valid `POST /api/characters` payload with `level: 5`, `ruleset: "SRD_2024"`, and `originFeatId`, **When** `CreateCharacterService.createCharacter` executes, **Then** it persists the character and its Level 1 class/subclass/origin feats with valid JSONB `choices` (`{}`) without duplicate key violations and returns HTTP 201 Created.
- **Given** a valid `POST /api/reference/items` payload from a `DUNGEON_MASTER` containing `mechanicalProperties` for `WEAPON`, `ARMOR`, or `CONSUMABLE`, **When** `ReferenceDataController.createItem` executes, **Then** Jackson deserializes the request without error, `ItemJpaEntity` persists and round-trips the JSONB `mechanical_properties` with its `"type"` discriminator, and the endpoint returns HTTP 201 Created.
- **Given** any unexpected database or runtime exception in `player-character-service`, **When** `GlobalExceptionHandler` intercepts it, **Then** it returns an RFC 7807 `ProblemDetail` JSON response containing `title`, `detail`, `status`, and `path`, and the frontend displays the detailed message in the toast notification.

---

## Defensive Security & Validation Considerations

- **JSONB Injection & Sanitization**: `CharacterFeatJpaEntity.fromDomain()` validates that `choicesJson` starts with `{` and ends with `}`, wrapping any plain string via Jackson `ObjectMapper` before binding to PostgreSQL `JSONB`.
- **Safe RFC 7807 Error Disclosure**: `GlobalExceptionHandler` logs the full stack trace server-side while returning a sanitized, human-readable pt-BR diagnostic message in `ProblemDetail.detail` without leaking raw SQL queries, database credentials, or internal stack frames.
- **Authentication & Authorization Impact**: `POST /api/reference/items` continues to enforce `X-Auth-User-Roles` containing `DUNGEON_MASTER` or `ADMIN`, and `POST /api/characters` enforces `X-Auth-User-Id` ownership. No secrets are hardcoded.

---

## Testing Strategy & Shift-Left Plan

| Scenario | Type | Target Test Class / File | Expected Outcome |
|---|---|---|---|
| Create character with Level 5 + `SRD_2024` + `originFeatId` | Unit | `CreateCharacterServiceTest.java` | `CharacterFeat.choicesJson` is valid JSON `{}` and `saveCharacterFeat` is not called redundantly after cascade save. |
| Create Homebrew Weapon, Armor & Consumable with `mechanicalProperties` | Unit | `HomebrewServiceTest.java` | Maps DTO to `WeaponMechanics` / `ArmorMechanics` / `ConsumableMechanics` and persists with `"type"` discriminator. |
| `DataIntegrityViolationException` & generic `Exception` handling | Unit | `GlobalExceptionHandlerTest.java` | Returns RFC 7807 `ProblemDetail` with status 409 / 500, `title`, `detail`, and `path`. |
| Frontend `apiFetch` error extraction | Unit | `lib/api/client.test.ts` | Displays `ProblemDetail.detail` or formatted status/path diagnostic message in toast. |

- **Verification Commands**:
  - `mvn -f player-character-service/backend/pom.xml test`
  - `npm --prefix dungeoneer-frontend run test:unit`

---

## Risks and Mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| Existing rows in `items` with `mechanical_properties` lacking `"type"` field | Medium | `ItemJpaEntity.deserializeMechanics()` infers type from `damageRolls` (`WEAPON`), `baseAc`/`armorCategory` (`ARMOR`), or `effectType` (`CONSUMABLE`) when `"type"` is absent. |
| Duplicate feat IDs when background default feat matches chosen `originFeatId` | Medium | `CreateCharacterService` tracks a `Set<UUID> addedFeatIds` so each `featId` is added at most once per character. |
