# Tasks: Add Encyclopedia Browse

> **Change ID:** `add-encyclopedia-browse`
> **Linked Proposal:** [proposal.md](./proposal.md)
> **Status:** in-progress
>
> All tasks must be `[x]` before this change can be archived via `sdd-spec-archive`.

---

## Implementation Tasks

- [ ] Verify dependency prerequisite: `add-main-dashboard` completed and archived
- [ ] Create Flyway migration `V7__create_spells_feats_items_and_fts_indexes.sql` with schema, GIN indexes, and cursor compound indexes
- [ ] Create seed JSON files for SRD 5.1/5.2 spells in `db/seed/` and implement `V8__seed_srd_spells.sql` (idempotent ON CONFLICT DO NOTHING)
- [ ] Create seed JSON files for SRD 5.1/5.2 feats in `db/seed/` and implement `V9__seed_srd_feats.sql` (idempotent ON CONFLICT DO NOTHING)
- [ ] Create seed JSON files for SRD 5.1/5.2 items in `db/seed/` and implement `V10__seed_srd_items.sql` (idempotent ON CONFLICT DO NOTHING)
- [ ] Create pure domain model `Item.java` in `domain/model/` with invariant checks
- [ ] Implement JPA entities (`SpellJpaEntity`, `FeatJpaEntity`, `ItemJpaEntity`) and Spring Data repositories with FTS & cursor queries
- [ ] Update `ReferenceDataRepository` port and `ReferenceDataPersistenceAdapter` with paginated query implementations
- [ ] Implement `CursorPageResponse<T>` DTO, category DTOs, use case extensions, and `ReferenceDataService`
- [ ] Add paginated REST endpoints with `@Min(1) @Max(100)` limit validation to `ReferenceDataController`
- [ ] Create frontend API client and TanStack `useInfiniteQuery` hooks in `lib/api/encyclopedia.ts`
- [ ] Build shared encyclopedia layout with category tabs and CC-BY 4.0 license footer
- [ ] Build search bar (300ms debounce), filter selectors, and infinite scroll sentinel components
- [ ] Implement browse pages for classes, spells, feats, items, and lineages with Dark Fantasy cards and skeletons

## Test Tasks

- [ ] Write Testcontainers integration test `SeedMigrationTest` verifying V7-V10 migrations, idempotency, counts, and FTS
- [ ] Write unit tests for domain model `Item.java` verifying invariant validations
- [ ] Write MockMvc tests in `ReferenceDataControllerTest` for cursor pagination, filtering, search, and limit validation
- [ ] Write frontend unit tests for encyclopedia API client and infinite query parameters
- [ ] Write frontend component tests for search debounce, filter controls, and empty states
- [ ] Execute full backend test suite (`mvn clean verify`) with >= 80% JaCoCo coverage
- [ ] Execute frontend unit tests (`npm run test:unit`) and production build (`npm run build`)

## Documentation Tasks

- [ ] Complete `proposal.md`, delta specifications, and task breakdown
- [ ] Maintain cumulative `walkthrough.md` in pt-BR documenting all phases, seed sample reports, and test results

## Review Gate

- [ ] All implementation tasks complete
- [ ] All tests passing
- [ ] Proposal acceptance criteria verified
- [ ] Ready for `sdd-spec-archive`
