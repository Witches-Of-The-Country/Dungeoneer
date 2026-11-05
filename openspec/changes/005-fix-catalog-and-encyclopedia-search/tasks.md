# Tasks: Fix Text Search in Item Catalog & All Encyclopedia Compendiums

> **Change ID:** `005-fix-catalog-and-encyclopedia-search`
> **Execution Order:** `005`
> **Linked Proposal:** [proposal.md](./proposal.md)
> **Status:** in-progress
>
> All tasks must be `[x]` before this change can be archived via `sdd-spec-archive`.

---

## Legacy & Query Adaptation Tasks

- [x] Update `SpringDataItemRepository.java` to support accent-insensitive, case-insensitive substring and prefix matching (`translate(lower(...))` + `LIKE`) across `name`, `name_en`, `description`, `item_type`, and `item_category`.
- [x] Update `SpringDataSpellRepository.java`, `SpringDataFeatRepository.java`, `SpringDataArchetypeRepository.java`, and `SpringDataLineageRepository.java` with the same accent-insensitive substring search logic.

## Implementation Tasks

- [x] Update `dungeoneer-frontend/components/character-sheet/item-catalog-modal.tsx` to normalize diacritics (`NFD`) in client filtering and avoid wiping items on partial keystrokes.
- [x] Verify `dungeoneer-frontend/components/encyclopedia/search-bar.tsx` and encyclopedia pages properly pass trimmed search queries.

## Test Tasks

- [x] Add repository/service tests in `player-character-service` verifying partial prefix queries (`"esp"`, `"arm"`) and unaccented queries (`"pocao"`, `"bencao"`).
- [x] Add frontend unit tests for `ItemCatalogModal` search input filtering.

## Documentation Tasks

- [x] Update delta specs in `specs/player-character-domain-delta.md` and `specs/frontend-web-app-delta.md`.

## Review Gate

- [x] All implementation tasks complete
- [x] All tests passing (`mvn clean test` and `npm test`)
- [x] Proposal acceptance criteria verified
- [ ] Ready for `sdd-spec-archive`
