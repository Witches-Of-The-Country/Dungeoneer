# Tasks: Add Encyclopedia Browse

> **Change ID:** `add-encyclopedia-browse`
> **Linked Proposal:** [proposal.md](./proposal.md)
> **Status:** in-progress
>
> All tasks must be `[x]` before this change can be archived via `sdd-spec-archive`.

---

## Implementation Tasks

- [x] Verify dependency prerequisite: `add-main-dashboard` completed and archived
- [x] Create Flyway migration `V7__create_spells_feats_items_and_fts_indexes.sql` with schema, GIN indexes, and cursor compound indexes
- [x] Create seed JSON files for SRD 5.1/5.2 spells in `db/seed/` and implement `V8__seed_srd_spells.sql` (idempotent ON CONFLICT DO NOTHING)
- [x] Create seed JSON files for SRD 5.1/5.2 feats in `db/seed/` and implement `V9__seed_srd_feats.sql` (idempotent ON CONFLICT DO NOTHING)
- [x] Create seed JSON files for SRD 5.1/5.2 items in `db/seed/` and implement `V10__seed_srd_items.sql` (idempotent ON CONFLICT DO NOTHING)
- [x] Create pure domain model `Item.java` in `domain/model/` with invariant checks
- [x] Implement JPA entities (`SpellJpaEntity`, `FeatJpaEntity`, `ItemJpaEntity`) and Spring Data repositories with FTS & cursor queries
- [x] Update `ReferenceDataRepository` port and `ReferenceDataPersistenceAdapter` with paginated query implementations
- [x] Implement `CursorPageResponse<T>` DTO, category DTOs, use case extensions, and `ReferenceDataService`
- [x] Add paginated REST endpoints with `@Min(1) @Max(100)` limit validation to `ReferenceDataController`
- [x] Create frontend API client and TanStack `useInfiniteQuery` hooks in `lib/api/encyclopedia.ts`
- [x] Build shared encyclopedia layout with category tabs and CC-BY 4.0 license footer
- [x] Build search bar (300ms debounce), filter selectors, and infinite scroll sentinel components
- [x] Implement browse pages for classes, spells, feats, items, and lineages with Dark Fantasy cards and skeletons

## Test Tasks

- [x] Write Testcontainers integration test `SeedMigrationTest` verifying V7-V10 migrations, idempotency, counts, and FTS
- [x] Write unit tests for domain model `Item.java` verifying invariant validations
- [x] Write MockMvc tests in `ReferenceDataControllerTest` for cursor pagination, filtering, search, and limit validation
- [x] Write frontend unit tests for encyclopedia API client and infinite query parameters
- [x] Write frontend component tests for search debounce, filter controls, and empty states
- [x] Execute full backend test suite (`mvn clean verify`) with >= 80% JaCoCo coverage
- [x] Execute frontend unit tests (`npm run test:unit`) and production build (`npm run build`)

## Documentation Tasks

- [x] Complete `proposal.md`, delta specifications, and task breakdown
- [x] Maintain cumulative `walkthrough.md` in pt-BR documenting all phases, seed sample reports, and test results

## Review Gate

- [x] All implementation tasks complete
- [x] All tests passing
- [x] Proposal acceptance criteria verified
- [x] Ready for `sdd-spec-archive`
