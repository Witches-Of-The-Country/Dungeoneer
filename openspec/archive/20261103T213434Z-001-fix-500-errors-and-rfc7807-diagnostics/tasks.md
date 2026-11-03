# Tasks: 001 — Fix HTTP 500 Errors on Character & Item Creation and Enhance RFC 7807 Diagnostics

> **Change ID:** `001-fix-500-errors-and-rfc7807-diagnostics`
> **Linked Proposal:** [proposal.md](./proposal.md)
> **Status:** completed
>
> All tasks must be `[x]` before this change can be archived via `sdd-spec-archive`.

---

## Implementation Tasks

- [x] Fix `CharacterFeat` instantiation in `CreateCharacterService.java` and `LevelUpService.java` so `choicesJson` is always valid JSON (`"{}"` or `{"source":"..."}`), deduplicate initial feats by `featId`, and remove the redundant `saveCharacterFeat` loop that caused duplicate constraint errors.
- [x] Add defensive JSON sanitization in `CharacterFeatJpaEntity.java` so non-JSON `choices` strings are wrapped as valid JSON objects before binding to PostgreSQL `JSONB`.
- [x] Update `CreateHomebrewItemRequest.java` and `ReferenceDataWebMapper.java` to deserialize `mechanicalProperties` via a structured DTO/mapper into `WeaponMechanics`, `ArmorMechanics`, `ConsumableMechanics`, or `EmptyItemMechanics` without requiring Jackson annotations on domain interfaces.
- [x] Update `ItemJpaEntity.java` so `serializeMechanics` writes the `"type"` discriminator (`"WEAPON"`, `"ARMOR"`, `"CONSUMABLE"`) and `deserializeMechanics` reconstructs the domain `ItemMechanics` accurately.
- [x] Update `GlobalExceptionHandler.java` in `player-character-service` with `@ExceptionHandler(DataIntegrityViolationException.class)` and fallback `@ExceptionHandler(Exception.class)` returning RFC 7807 `ProblemDetail` with actionable diagnostics.
- [x] Update `dungeoneer-frontend/lib/api/client.ts` to extract and display detailed RFC 7807 and fallback server error messages clearly in pt-BR.

## Test Tasks

- [x] Add unit tests in `CreateCharacterServiceTest.java`, `HomebrewServiceTest.java`, and `GlobalExceptionHandlerTest.java` verifying valid JSON `choicesJson`, Homebrew item mechanics mapping/serialization, and RFC 7807 fallback error responses.
- [x] Update `dungeoneer-frontend/lib/api/client.test.ts` to verify improved error message extraction.

## Documentation Tasks

- [x] Document the updated error handling and serialization contracts in `specs/player-character-domain-delta.md`.

## Review Gate

- [x] All implementation tasks complete
- [x] All tests passing
- [x] Proposal acceptance criteria verified
- [x] Ready for `sdd-spec-archive`
