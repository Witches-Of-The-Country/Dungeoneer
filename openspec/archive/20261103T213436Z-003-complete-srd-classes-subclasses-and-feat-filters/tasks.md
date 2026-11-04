# Tasks: 003 — Complete SRD 2014 & 2024 Class Feats, Subclasses & Encyclopedia Multi-Select Feat Filter

> **Change ID:** `003-complete-srd-classes-subclasses-and-feat-filters`
> **Linked Proposal:** [proposal.md](./proposal.md)
>
> All tasks must be `[x]` before this change can be archived via `sdd-spec-archive`.

---

## Implementation Tasks

- [x] Create Flyway migration `V20__seed_complete_srd_classes_and_subclasses_progression.sql` populating Level 1–20 `archetype_feats` for all 13 classes, additional subclasses per class, and complete `subclass_feats` across `SRD_2014` and `SRD_2024`.
- [x] Update `ReferenceDataController.java`, `GetReferenceDataUseCase.java`, `ReferenceDataService.java`, `ReferenceDataPersistenceAdapter.java`, and `SpringDataFeatRepository.java` to filter `GET /api/reference/feats` by `List<String> featTypes`.
- [x] Update `dungeoneer-frontend/lib/api/encyclopedia.ts` and `dungeoneer-frontend/app/encyclopedia/feats/page.tsx` with a multi-select filter bar for `ORIGIN`, `GENERAL`, `CLASS`, `SUBCLASS`, and `LINEAGE` feats (and display the feat type badge on each card).

## Test Tasks

- [x] Add unit tests in `ReferenceDataControllerTest` / `ReferenceDataServiceTest` and `dungeoneer-frontend/__tests__/encyclopedia-browse.test.tsx` verifying multi-select `featTypes` filtering.

## Documentation Tasks

- [x] Document the `featTypes` query parameter and complete SRD progression seed coverage in `specs/player-character-data-model-delta.md` and `specs/frontend-web-app-delta.md`.

## Review Gate

- [x] All implementation tasks complete
- [x] All tests passing
- [x] Proposal acceptance criteria verified
- [x] Ready for `sdd-spec-archive`
