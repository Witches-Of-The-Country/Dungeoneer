# Tasks: 007-srd-class-progression-feats-and-encyclopedia-multiselect-filters

## Phase 1: Database Verification & Progression Feats
- [x] 1.1 Verify existing `V20` progression feats in `player_character.archetype_feats` and `player_character.subclass_feats` for levels 1 to 20 across all 12 classes.
- [x] 1.2 If any class progression feats or subclass progression feats are missing, draft Flyway migration `V25__seed_missing_srd_class_and_subclass_progression_feats.sql` (Verified: V20 already seeded all SRD classes and subclasses progression).
- [x] 1.3 Link feats to `archetype_feats` and `subclass_feats` tables with appropriate `unlock_level`.

## Phase 2: Backend Repository & Controller Multi-Select Filter Support
- [x] 2.1 Update `SpringDataItemRepository.java`:
  - Support multi-category filtering (`ANY(string_to_array(:categoriesStr, ','))`).
  - Match category against `i.item_category` OR `i.item_type` for types like `Poção`, `Item Maravilhoso`, `Equipamento de Aventura`.
  - Support multi-rarity filtering (`ANY(string_to_array(:raritiesStr, ','))`).
- [x] 2.2 Update `SpringDataSpellRepository.java`:
  - Support multi-level filtering (`levelsStr` array matching).
  - Support multi-school filtering (`schoolsStr` array matching).
- [x] 2.3 Update `ReferenceDataController.java` to accept list query parameters for spells and items.
- [x] 2.4 Add repository and controller tests verifying multi-select OR behavior.

## Phase 3: Frontend Item Filters Defect Fixes (Rarity & Category)
- [x] 3.1 In `app/encyclopedia/items/page.tsx`, map `ITEM_RARITIES` values to uppercase English (`COMMON`, `UNCOMMON`, `RARE`, `VERY_RARE`, `LEGENDARY`, `ARTIFACT`).
- [x] 3.2 Align `ITEM_CATEGORIES` query parameters to match backend item types (`POTION`, `RING`, `ROD`, `SCROLL`, `STAFF`, `WAND`, `WONDROUS`, `ADVENTURING_GEAR`).

## Phase 4: Frontend Multi-Select Filter Components with OR Logic
- [x] 4.1 Update `encyclopedia.ts` filter interfaces (`SpellFilters`, `ItemFilters`) to support string arrays for filter options.
- [x] 4.2 In `app/encyclopedia/spells/page.tsx`, replace single dropdowns with multi-select chip toggle bars for Level (0-9) and School.
- [x] 4.3 In `app/encyclopedia/items/page.tsx`, replace single dropdowns with multi-select chip toggle bars for Category and Rarity.
- [x] 4.4 Verify Feats, Classes, and Lineages pages properly handle multi-select filters.

## Phase 5: Rich Feat Selector in Level Up Wizard
- [x] 5.1 In `components/character-sheet/level-up-wizard-modal.tsx`, replace the `<select>` dropdown in the Feat step with a rich Feat Selector interface.
- [x] 5.2 Implement search input (client-side debounced search matching feat name, description, and prerequisites).
- [x] 5.3 Implement category filter chips (Todos, Origem, Geral, Estilo de Luta, Épico, Homebrew).
- [x] 5.4 Implement pagination (e.g. 5-6 feats per page) with previous/next controls, page indicator, and total count.
- [x] 5.5 Implement "Ver Detalhes" dialog/pop-up showing complete feat mechanics, requirements, and description.
- [x] 5.6 Add visual selection indicator (border highlight, checkmark, selected summary badge).

## Phase 6: Verification & Testing
- [x] 6.1 Run backend tests (`mvn test`).
- [x] 6.2 Run frontend unit tests (`npm run test:unit`).
- [x] 6.3 Run frontend production build (`npm run build`).

## Review Gate
- [x] All 6 acceptance criteria validated.
- [x] Filtering items by "Incomum" or "Raro" displays matching items.
- [x] Filtering items by "Poções" or "Equipamento de Aventura" displays matching items.
- [x] Multi-select OR filters on spells and items function smoothly.
- [x] Rich Feat Selector in Level Up Wizard allows searching, paginating, viewing details, and selecting feats.
- [x] Conventional commit prepared for user approval.
