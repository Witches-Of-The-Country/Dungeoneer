# Tasks: 008-srd-item-catalog-seed-and-category-normalization

## Phase 1: Database Migration & Category Normalization
- [x] 1.1 Create Flyway migration `V24__normalize_srd_item_categories_and_seed_catalog.sql` in `player-character-service/backend/src/main/resources/db/migration/`
- [x] 1.2 Write SQL `UPDATE` statements to normalize existing `player_character.items` categories based on `item_type` (POTION, WONDROUS, RING, SCROLL, WAND, ROD, STAFF, EQUIPMENT)
- [x] 1.3 Write SQL `INSERT` statements with `ON CONFLICT (slug) DO UPDATE` to seed all iconic SRD potions (Healing tiers, Speed, Invisibility, Giant Strength variants, etc.)
- [x] 1.4 Write SQL `INSERT` statements to seed all iconic SRD wondrous items (Bag of Holding, Cloak of Protection, Boots of Elvenkind, Amulet of Health, etc.)
- [x] 1.5 Write SQL `INSERT` statements to seed all iconic SRD rings, scrolls, wands, rods, and staves
- [x] 1.6 Write SQL `INSERT` statements to seed standard SRD adventuring gear, equipment packs, tool sets, and focuses
- [x] 1.7 Run `mvn test` in `player-character-service` to verify migration syntax and database integration

## Phase 2: Backend Query Normalization & Rarity Filter Fix
- [x] 2.1 Update `SpringDataItemRepository.java` to support category matching against both `item_category` and `item_type`
- [x] 2.2 Update `SpringDataItemRepository.java` to properly match rarity enum strings, treating `COMMON` to include non-magical/`NONE`/`NULL` items
- [x] 2.3 Add unit / integration tests in `ReferenceDataServiceTest` verifying category filters (`POTION`, `WONDROUS`, `EQUIPMENT`) and rarity filters (`COMMON`, `RARE`, `VERY_RARE`)
- [x] 2.4 Verify all tests pass with `mvn test`

## Phase 3: Frontend Catalog & Store Filters Alignment
- [x] 3.1 Update `app/encyclopedia/items/page.tsx` rarity filter to map cleanly to uppercase backend enums (`COMMON`, `UNCOMMON`, `RARE`, `VERY_RARE`, `LEGENDARY`, `ARTIFACT`)
- [x] 3.2 Update `app/encyclopedia/items/page.tsx` category tabs to include canonical categories (`ALL`, `WEAPON`, `ARMOR`, `POTION`, `WONDROUS`, `RING`, `SCROLL`, `WAND`, `ROD`, `STAFF`, `EQUIPMENT`)
- [x] 3.3 Update `components/character-sheet/item-catalog-modal.tsx` category and rarity selectors to match encyclopedia categories
- [x] 3.4 Run `npm run test:unit` in `dungeoneer-frontend` to verify all components and modals render without regression
- [x] 3.5 Run `npm run build` in `dungeoneer-frontend` to ensure TypeScript compilation

## Phase 4: Container Build & End-to-End Verification
- [x] 4.1 Rebuild and restart `player-character-service` via Docker Compose (`docker --context default compose up -d --build player-character-service`)
- [x] 4.2 Verify items query in backend returns non-zero results for Potions, Wondrous, and Equipment
- [x] 4.3 Verify item detail modal opens smoothly for seeded items

## Review Gate
- [x] All Flyway migration statements execute idempotently with zero errors.
- [x] Backend test suite passes (`mvn test`) with ≥80% line coverage on updated logic.
- [x] Frontend unit tests pass (`npm run test:unit`) with zero regressions.
- [x] Ready-to-use Conventional Commit commands provided for each module and root repository.
