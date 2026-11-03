Focus: Backend / Spring Boot 4 (Java 21) / Flyway Database Migration / PostgreSQL 15
Task: Execute Phase 1: Database Migration & SRD Item Category Normalization

## Execution Context
- Active Plan: `openspec/changes/008-srd-item-catalog-seed-and-category-normalization/tasks.md`
- Proposal Reference: `openspec/changes/008-srd-item-catalog-seed-and-category-normalization/proposal.md`
- Data Model Delta: `openspec/changes/008-srd-item-catalog-seed-and-category-normalization/specs/player-character-data-model-delta.md`
- Scope: Execute Phase 1 only (Tasks 1.1 to 1.7). Validate Flyway migration V24, verify SQL normalization and seed statements, run database migration tests, and initiate the cumulative walkthrough. Do not proceed to subsequent phases.

## Implementation Steps
1. **Verify Flyway Migration V24:**
   - Inspect `player-character-service/backend/src/main/resources/db/migration/V24__normalize_srd_item_categories_and_seed_catalog.sql`.
   - Ensure SQL `UPDATE` statements accurately map existing items where `item_category = 'MAGIC'` to canonical categories (`POTION`, `WONDROUS`, `RING`, `SCROLL`, `WAND`, `ROD`, `STAFF`, `EQUIPMENT`) based on `item_type`.
   - Ensure standard non-magical equipment and adventuring gear are normalized to `EQUIPMENT` with `rarity = 'COMMON'`.
2. **Verify Idempotent Seed Data:**
   - Confirm missing iconic SRD 2014 & SRD 2024 items (healing potions across all tiers, utility potions, wondrous items, rings, scrolls, wands, rods, staves, and adventuring gear) are seeded with `ON CONFLICT (ruleset, slug) DO UPDATE`.
3. **Execute Backend Migration & Integration Tests:**
   - Run Maven test suite on `player-character-service` backend to ensure PostgreSQL test containers apply migration `V24` with zero SQL errors or schema violations.

## Quality Gates & Verification
- Run local validation:
  ```bash
  mvn -f player-character-service/backend/pom.xml test
  ```
- Commit locally:
  ```bash
  git commit -m "feat(catalog): normalize srd item categories and seed catalog via migration V24"
  ```

## Phase Handoff & Cumulative Walkthrough
Create the shared walkthrough artifact (`openspec/changes/008-srd-item-catalog-seed-and-category-normalization/walkthrough.md` in pt-BR) recording execution progress:
1. List of files created/modified in this phase.
2. Verification and test results (Flyway migration output and test execution).
3. Handoff context prepared for Phase 2 (Backend Query Normalization & Rarity Filter Unit Tests).
Pause and prompt the user to commit progress before proceeding to Phase 2.
