# Proposal: 007-srd-class-progression-feats-and-encyclopedia-multiselect-filters

## Summary
Complete the SRD class progression features and subclass progression features across all 12 classes for character levels 1 to 20, transform all Encyclopedia filters (Spells, Items, Feats, Classes, Lineages) into multi-select controls with OR logic, and fix the defects in Item Rarity and Category filtering.

## Motivation
Currently:
1. Only early levels (levels 1-3, and sparse levels up to 5) have seeded class and subclass progression features in the database. Leveling up beyond level 5 (or level 2/3 for most classes) does not unlock any class or subclass feats.
2. In `/encyclopedia/items`, the rarity filter is non-functional because the frontend sends Portuguese strings (`"Comum"`, `"Incomum"`, `"Raro"`) while the database stores English uppercase enums (`'COMMON'`, `'UNCOMMON'`, `'RARE'`).
3. In `/encyclopedia/items`, the category filter only returns results for `"Armas"` (`WEAPON`) and `"Armaduras"` (`ARMOR`). All other items (potions, wondrous items, rings, scrolls, adventuring gear) are categorized as `'MAGIC'` in `item_category` with their specific type in `item_type`, causing category filters to return 0 results.
4. Encyclopedia filters on Spells and Items are single-select dropdowns that do not permit combining multiple levels, schools, categories, or rarities with OR logic.

## Scope

### In Scope
- Database migration (`V24`) seeding complete SRD 2014/2024 class progression features and subclass progression features for all 12 classes from levels 1 to 20, linked to `archetype_feats` and `subclass_feats`.
- Fixing the Item Rarity filter by aligning frontend values with database enum strings (`'COMMON'`, `'UNCOMMON'`, `'RARE'`, `'VERY_RARE'`, `'LEGENDARY'`, `'ARTIFACT'`).
- Fixing the Item Category filter by supporting both `item_category` and `item_type` matching or migrating item categories to standard values (`WEAPON`, `ARMOR`, `POTION`, `RING`, `ROD`, `SCROLL`, `STAFF`, `WAND`, `WONDROUS`, `ADVENTURING_GEAR`).
- Upgrading Spells filters to multi-select chip/toggle controls with OR logic for spell levels (0 to 9), magic schools, and ruleset.
- Upgrading Items filters to multi-select chip/toggle controls with OR logic for categories, rarities, and ruleset.
- Upgrading Feats, Classes, and Lineages filters to multi-select controls where applicable.
- Backend repository and controller updates to accept collections or comma-separated lists for filtered properties and query them using PostgreSQL `ANY(string_to_array(...))` or JPA `IN (...)`.

### Out of Scope
- Adding homebrew classes or homebrew subclasses outside SRD 2014/2024.
- Automated spellbook auto-learning mechanics for non-prepared casters.

## Proposed Solution
1. **Migration V24**: Seed all remaining SRD class features (e.g., Barbarian Extra Attack, Brutal Critical, Rage improvements; Fighter Indomitable, Extra Attack (2) and (3); Rogue Uncanny Dodge, Evasion, Reliable Talent, Blindsense, Slippery Mind, Elusive, Stroke of Luck; Paladin Auras; Ranger features; Monk Ki/Focus progression, Diamond Body, Empty Body; Druid Wild Shape enhancements, Timeless Body; Cleric Divine Intervention, Destroy Undead; Bard Font of Inspiration, Magical Secrets; Wizard Spell Mastery, Signature Spells; Warlock Mystic Arcanum; Sorcerer Metamagic and Sorcerous Restoration) and all subclass progression milestones for all official subclasses.
2. **Item Filters Bug Fixes**:
   - Update `app/encyclopedia/items/page.tsx` `ITEM_RARITIES` array to use English backend values (`COMMON`, `UNCOMMON`, etc.) while keeping Portuguese labels (`"Comum"`, `"Incomum"`).
   - In `SpringDataItemRepository.java`, update the SQL query so category filtering matches against either `i.item_category` or `i.item_type` (case-insensitive / mapped enum), allowing filters like `POTION` or `ADVENTURING_GEAR` to match records correctly.
3. **Multi-Select Filters with OR Logic**:
   - In backend controllers (`ReferenceDataController`) and repositories (`SpringDataSpellRepository`, `SpringDataItemRepository`, `SpringDataFeatRepository`), accept list parameters (`levels`, `schools`, `categories`, `rarities`, `rulesets`) and filter using `ANY(string_to_array(:param, ','))` or `IN (...)`.
   - In frontend pages (`/encyclopedia/spells`, `/encyclopedia/items`), replace single `<select>` with multi-select badge/chip filter bars that toggle values in an array and trigger TanStack Query refetches with OR semantics.

### Impacted Files

| File | Status | Layer | Description |
|------|--------|-------|-------------|
| `V24__seed_complete_srd_class_and_subclass_progression_feats.sql` | Created | Database | Seed progression feats and milestones for all 12 classes (levels 1-20) |
| `SpringDataItemRepository.java` | Modified | Adapter Out | Multi-select OR query for item categories and rarities; fix category/type matching |
| `SpringDataSpellRepository.java` | Modified | Adapter Out | Multi-select OR query for spell levels and schools |
| `ReferenceDataController.java` | Modified | Adapter In | Accept list parameters for item and spell filters |
| `ReferenceDataService.java` | Modified | Application | Pass filter lists to repositories |
| `encyclopedia.ts` | Modified | Frontend | Update query filters to accept arrays of strings/numbers |
| `app/encyclopedia/items/page.tsx` | Modified | Frontend | Multi-select chip filters, correct rarity enum values, and category mapping |
| `app/encyclopedia/spells/page.tsx` | Modified | Frontend | Multi-select chip filters for spell levels and schools with OR logic |
| `app/encyclopedia/feats/page.tsx` | Modified | Frontend | Ensure multi-select chip filters use OR logic for types and rulesets |

## Defensive Security & Validation Considerations
- **SQL Injection Prevention**: All multi-select filtering uses parameterized queries (`ANY(string_to_array(:levelsStr, ','))` or Spring Data `IN (:list)`); no dynamic string concatenation in SQL queries.
- **Input Sanitization**: Query parameter arrays are bounded (e.g. spell levels limited to integers 0-9; rarity and school strings validated against allowed enums).

## Testing Strategy & Shift-Left Plan
- **Database & Migration Tests**:
  - Verify migration `V24` executes cleanly without constraint violations.
  - Test `ReferenceDataRepository.findProgressionFeats` for level 5, 11, and 20 across all classes.
- **Repository Unit & Integration Tests**:
  - Test `SpringDataItemRepository` filtering by multiple rarities (`RARE,VERY_RARE`) returns items matching either rarity (OR logic).
  - Test `SpringDataItemRepository` filtering by `POTION` returns potions seeded with `item_category = 'MAGIC'` and `item_type = 'Poção'`.
  - Test `SpringDataSpellRepository` filtering by levels `[1, 2, 3]` returns spells from any of those levels.
- **Frontend Component Tests**:
  - Verify toggling multiple filter chips updates query parameters and displays expected items.

## Risks and Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| Large V24 migration script payload | Low | Script uses standard PostgreSQL batch `INSERT` statements with `ON CONFLICT DO NOTHING`. |
| Breaking existing single-select query callers | Low | Backend query methods accept both single and comma-separated lists gracefully. |

## Acceptance Criteria (Given-When-Then)

### AC 1: High-Level Class and Subclass Progression Feats
- **Given** a character advancing past level 3 up to level 20 in any of the 12 SRD classes,
- **When** the level-up process executes,
- **Then** all class progression feats and subclass progression feats designated for that level are automatically unlocked and present in `findProgressionFeats`.

### AC 2: Working Item Rarity Filter
- **Given** a user viewing `/encyclopedia/items`,
- **When** the user selects a rarity filter such as "Incomum" or "Raro",
- **Then** the page sends the matching backend enum (`UNCOMMON` or `RARE`) and displays items of that rarity instead of returning an empty list.

### AC 3: Working Item Category Filter
- **Given** a user viewing `/encyclopedia/items`,
- **When** the user filters by "Poções", "Itens Maravilhosos", or "Equipamento de Aventura",
- **Then** items belonging to those subcategories/types are returned and displayed.

### AC 4: Multi-Select Spell Filters with OR Logic
- **Given** a user viewing `/encyclopedia/spells`,
- **When** the user selects multiple levels (e.g. Level 1 and Level 2) and/or multiple schools (e.g. Evocação and Necromancia),
- **Then** the spells displayed include any spell matching level 1 OR 2, and matching Evocação OR Necromancia.

### AC 5: Multi-Select Item Filters with OR Logic
- **Given** a user viewing `/encyclopedia/items`,
- **When** the user selects multiple rarities (e.g. Comum e Incomum) or multiple categories (e.g. Armas e Armaduras),
- **Then** the items displayed include items matching any of the selected rarities and any of the selected categories.
