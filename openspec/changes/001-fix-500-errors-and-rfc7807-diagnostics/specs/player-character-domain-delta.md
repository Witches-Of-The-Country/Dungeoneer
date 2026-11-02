# Delta Spec: `player-character-domain`

> **Change ID:** `001-fix-500-errors-and-rfc7807-diagnostics`
> **Target Canonical Spec:** `openspec/specs/player-character-domain.md`
> **Change Type:** Modify

## Architectural Updates

### 1. `CharacterFeat` JSONB Choices & Cascade Persistence
- `CharacterFeat.choicesJson` MUST always contain a valid JSON object string (default `"{}"`).
- `CreateCharacterService` and `LevelUpService` MUST NOT pass raw enum names (`"ORIGIN"`, `"CLASS"`, `"SUBCLASS"`) into `choicesJson`, and MUST rely on single-pass persistence (or deduplicated `featId` set) to respect `uq_character_feat (character_id, feat_id)`.

### 2. Homebrew `ItemMechanics` Web Boundary Mapping & JSONB Discriminator
- `CreateHomebrewItemRequest` MUST accept `mechanicalProperties` as a structured boundary DTO (`HomebrewItemMechanicsRequest`) rather than exposing the unannotated domain sealed interface `ItemMechanics` directly to Jackson.
- `ItemJpaEntity` MUST include `"type": "WEAPON" | "ARMOR" | "CONSUMABLE"` when serializing `ItemMechanics` to the `items.mechanical_properties` JSONB column.

### 3. RFC 7807 `ProblemDetail` Fallback Handler
- `GlobalExceptionHandler` MUST handle `DataIntegrityViolationException` (HTTP 409 Conflict / 400 Bad Request) and `Exception` (HTTP 500 Internal Server Error) and return a structured `ProblemDetail` with `title`, `detail`, `status`, and `path`.
