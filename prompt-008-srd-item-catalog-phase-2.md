Focus: Backend / Spring Boot 4 (Java 21) / Clean Architecture / Spring Data JPA
Task: Execute Phase 2: Backend Query Normalization & Rarity Filter Fix

## Execution Context
- Active Plan: `openspec/changes/008-srd-item-catalog-seed-and-category-normalization/tasks.md`
- Proposal Reference: `openspec/changes/008-srd-item-catalog-seed-and-category-normalization/proposal.md`
- Scope: Execute Phase 2 only (Tasks 2.1 to 2.4). Normalize repository query filtering for categories and rarity, add unit/integration tests in `ReferenceDataServiceTest`, verify the test suite passes, and append execution progress to the cumulative walkthrough. Do not proceed to subsequent phases.

## Implementation Steps
1. **Update Item Query Normalization in Repository:**
   - Update `SpringDataItemRepository` custom query to support dual matching on category: match against `item_category` or match substring against `item_type` (case-insensitive) so that canonical category filters accurately capture both normalized records and existing sub-typed items.
   - Update rarity filtering logic to handle uppercase rarity enums, ensuring that filtering by `COMMON` also matches items with `rarity IS NULL` or `rarity = 'NONE'` (non-magical equipment).
2. **Add Comprehensive Filter Tests in ReferenceDataServiceTest:**
   - Add unit/integration test cases to verify category filtering across canonical types (e.g., `POTION`, `WONDROUS`, `EQUIPMENT`).
   - Add test cases to verify rarity filtering across tiers (e.g., `COMMON`, `RARE`, `VERY_RARE`), ensuring that `COMMON` correctly includes mundane items.
   - Maintain high test coverage following the project's TDD and Clean Architecture conventions.
3. **Execute Verification:**
   - Run the Maven test suite on `player-character-service` backend to ensure all existing and new tests pass without regressions.

## Quality Gates & Verification
- Run local validation:
  ```bash
  mvn -f player-character-service/backend/pom.xml test
  ```
- Commit locally:
  ```bash
  git commit -m "feat(backend): normalize item repository category and rarity filters with tests"
  ```

## Phase Handoff & Cumulative Walkthrough
Update the shared cumulative walkthrough artifact (`openspec/changes/008-srd-item-catalog-seed-and-category-normalization/walkthrough.md` in pt-BR) appending Phase 2 execution details:
1. List of files created/modified in this phase.
2. Verification and test results (test execution summary and line coverage).
3. Handoff context prepared for Phase 3 (Frontend Catalog & Store Filters Alignment).
Pause and prompt the user to commit progress before proceeding to Phase 3.
