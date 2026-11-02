# Tasks: 007-srd-class-progression-feats-and-encyclopedia-multiselect-filters

## Phase 1: Database Migration V24 (Class & Subclass Progression Feats Seed)
- [ ] 1.1 Draft Flyway migration `V24__seed_complete_srd_class_and_subclass_progression_feats.sql` containing all SRD class features for all 12 classes (Barbarian, Bard, Cleric, Druid, Fighter, Monk, Paladin, Ranger, Rogue, Sorcerer, Warlock, Wizard) for levels 1 to 20.
- [ ] 1.2 Include subclass progression milestones for official subclasses across all tiers (levels 3, 6, 7, 10, 11, 14, 15, 17, 18, 20).
- [ ] 1.3 Link feats to `archetype_feats` and `subclass_feats` tables with appropriate `unlock_level`.
- [ ] 1.4 Test migration V24 locally against PostgreSQL container.

## Phase 2: Backend Repository & Controller Multi-Select Filter Support
- [ ] 2.1 Update `SpringDataItemRepository.java`:
  - Support multi-category filtering (`ANY(string_to_array(:categoriesStr, ','))`).
  - Match category against `i.item_category` OR `i.item_type` for types like `Poção`, `Item Maravilhoso`, `Equipamento de Aventura`.
  - Support multi-rarity filtering (`ANY(string_to_array(:raritiesStr, ','))`).
- [ ] 2.2 Update `SpringDataSpellRepository.java`:
  - Support multi-level filtering (`levelsStr` array matching).
  - Support multi-school filtering (`schoolsStr` array matching).
- [ ] 2.3 Update `ReferenceDataController.java` to accept list query parameters for spells and items.
- [ ] 2.4 Add repository and controller tests verifying multi-select OR behavior.

## Phase 3: Frontend Item Filters Defect Fixes (Rarity & Category)
- [ ] 3.1 In `app/encyclopedia/items/page.tsx`, map `ITEM_RARITIES` values to uppercase English (`COMMON`, `UNCOMMON`, `RARE`, `VERY_RARE`, `LEGENDARY`, `ARTIFACT`).
- [ ] 3.2 Align `ITEM_CATEGORIES` query parameters to match backend item types (`POTION`, `RING`, `ROD`, `SCROLL`, `STAFF`, `WAND`, `WONDROUS`, `ADVENTURING_GEAR`).

## Phase 4: Frontend Multi-Select Filter Components with OR Logic
- [ ] 4.1 Update `encyclopedia.ts` filter interfaces (`SpellFilters`, `ItemFilters`) to support string arrays for filter options.
- [ ] 4.2 In `app/encyclopedia/spells/page.tsx`, replace single dropdowns with multi-select chip toggle bars for Level (0-9) and School.
- [ ] 4.3 In `app/encyclopedia/items/page.tsx`, replace single dropdowns with multi-select chip toggle bars for Category and Rarity.
- [ ] 4.4 Verify Feats, Classes, and Lineages pages properly handle multi-select filters.

## Phase 5: Verification & Testing
- [ ] 5.1 Run backend tests (`mvn test`).
- [ ] 5.2 Run frontend unit tests (`npm run test:unit`).
- [ ] 5.3 Run frontend production build (`npm run build`).

## Review Gate
- [ ] All 5 acceptance criteria validated.
- [ ] Migration V24 executes without errors on restart.
- [ ] Filtering items by "Incomum" or "Raro" displays matching items.
- [ ] Filtering items by "Poções" or "Equipamento de Aventura" displays matching items.
- [ ] Multi-select OR filters on spells and items function smoothly.
- [ ] Conventional commit prepared for user approval.
