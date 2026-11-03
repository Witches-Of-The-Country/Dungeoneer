# Proposal: 008-srd-item-catalog-seed-and-category-normalization

## Summary
Seed the complete SRD 2014 & 2024 items catalog (all missing Potions, Wondrous Items, Rings, Scrolls, Wands, Rods, Staves, and Adventuring Gear), normalize item categories in the PostgreSQL database from generic `'MAGIC'` to canonical categories (`POTION`, `WONDROUS`, `RING`, `SCROLL`, `WAND`, `ROD`, `STAFF`, `EQUIPMENT`), and fix the category and rarity filter query bugs across both the backend JPA repository and the frontend item catalog and inventory store interfaces.

## Motivation
1. In the Item Encyclopedia (`/encyclopedia/items`) and the Item Catalog & Store modal (`ItemCatalogModal`), selecting categories like **Poções**, **Itens Maravilhosos**, or **Equipamento** returns zero items. This occurs because in `V13__seed_complete_srd_items.sql`, non-weapon and non-armor items were stored with `item_category = 'MAGIC'`, while their subcategory (e.g. `Poção`, `Item Maravilhoso`, `Anel`, `Pergaminho`) was stored in `item_type`.
2. The rarity filter is non-functional because the frontend sends Portuguese strings (`"Comum"`, `"Incomum"`, `"Raro"`, etc.) or inconsistent casing, while the backend database stores English uppercase enums (`'COMMON'`, `'UNCOMMON'`, `'RARE'`, `'VERY_RARE'`, `'LEGENDARY'`, `'ARTIFACT'`). Items without magical rarity often have `rarity IS NULL`, `'NONE'` or `'COMMON'`.
3. Essential adventuring gear (backpack, bedroll, rations, waterskin, rope, torches, thieves' tools, herbalism kit, artisan's tools, holy symbols, arcane focuses) and iconic SRD magic items (Healing potions of all tiers, Bag of Holding, Cloak of Protection, Ring of Protection, Boots of Elvenkind, Wand of Magic Missiles, etc.) are partially or entirely missing from the seed database.

## Scope

### In Scope
- Migration `V24` normalizing existing items in `player_character.items` based on `item_type`.
- Seeding complete SRD 2014 & 2024 catalog for Potions, Wondrous Items, Rings, Scrolls, Wands, Rods, Staves, and Adventuring Gear.
- Backend filtering fixes in `SpringDataItemRepository` for dual category/type matching and rarity normalization.
- Frontend category and rarity alignment in `/encyclopedia/items` and `ItemCatalogModal`.

### Out of Scope
- Adding homebrew classes or feats.
- Modifying character inventory association schemas (character item purchase and hydration is already working).

## Proposed Solution

### Technical Approach
1. **Migration `V24__normalize_srd_item_categories_and_seed_catalog.sql`**:
   - Executes SQL `UPDATE` to convert `item_category = 'MAGIC'` into canonical values:
     - `item_type ILIKE '%poção%' OR item_type ILIKE '%potion%'` -> `item_category = 'POTION'`
     - `item_type ILIKE '%anel%' OR item_type ILIKE '%ring%'` -> `item_category = 'RING'`
     - `item_type ILIKE '%pergaminho%' OR item_type ILIKE '%scroll%'` -> `item_category = 'SCROLL'`
     - `item_type ILIKE '%varinha%'` or `%wand%` -> `item_category = 'WAND'`
     - `item_type ILIKE '%bastão%'` or `%rod%` -> `item_category = 'ROD'`
     - `item_type ILIKE '%cajado%'` or `%staff%` -> `item_category = 'STAFF'`
     - `item_type ILIKE '%maravilhoso%'` or `%wondrous%` -> `item_category = 'WONDROUS'`
     - Adventuring gear and tools -> `item_category = 'EQUIPMENT'`, `rarity = 'COMMON'`
   - Inserts all missing standard SRD 2014 & 2024 items with `ON CONFLICT (slug) DO UPDATE` to ensure idempotency.
2. **Backend Repository Normalization (`SpringDataItemRepository.java`)**:
   - Allows category matching against both `item_category` and `item_type`:
     ```sql
     AND (CAST(:itemCategory AS VARCHAR) IS NULL
          OR UPPER(i.item_category) = UPPER(CAST(:itemCategory AS VARCHAR))
          OR UPPER(i.item_type) ILIKE CONCAT('%', UPPER(CAST(:itemCategory AS VARCHAR)), '%'))
     ```
   - Normalizes rarity matching to treat `COMMON` as matching `COMMON`, `NONE`, or `NULL`:
     ```sql
     AND (CAST(:rarity AS VARCHAR) IS NULL
          OR UPPER(i.rarity) = UPPER(CAST(:rarity AS VARCHAR))
          OR (UPPER(CAST(:rarity AS VARCHAR)) = 'COMMON' AND (i.rarity IS NULL OR UPPER(i.rarity) = 'NONE')))
     ```
3. **Frontend UI Filter Alignment**:
   - Updates `app/encyclopedia/items/page.tsx` and `components/character-sheet/item-catalog-modal.tsx` to use canonical category keys (`WEAPON`, `ARMOR`, `POTION`, `WONDROUS`, `RING`, `SCROLL`, `WAND`, `ROD`, `STAFF`, `EQUIPMENT`) and uppercase rarity enum values (`COMMON`, `UNCOMMON`, `RARE`, `VERY_RARE`, `LEGENDARY`, `ARTIFACT`).

### Impacted Files

| File | Operation | Layer | Description |
|------|-----------|-------|-------------|
| `V24__normalize_srd_item_categories_and_seed_catalog.sql` | CREATE | Database (Flyway) | Normalizes existing item categories and seeds full SRD catalog |
| `SpringDataItemRepository.java` | MODIFY | Backend Adapter Out | Supports dual category/type matching and rarity normalization |
| `ReferenceDataServiceTest.java` | MODIFY | Backend Test | Adds integration tests for category and rarity filtering |
| `app/encyclopedia/items/page.tsx` | MODIFY | Frontend Page | Canonical category tabs and uppercase rarity filter keys |
| `components/character-sheet/item-catalog-modal.tsx` | MODIFY | Frontend Component | Aligns category dropdown and rarity dropdown with backend enums |

## Acceptance Criteria

- **AC1 (Category Normalization)**: Given existing items in `player_character.items` with `item_category = 'MAGIC'`, When migration `V24` executes, Then their `item_category` is updated to canonical values (`POTION`, `WONDROUS`, `RING`, `SCROLL`, `WAND`, `ROD`, `STAFF`, `EQUIPMENT`) according to `item_type`.
- **AC2 (Seed Idempotency)**: Given a database with or without previous seeds, When migration `V24` is applied, Then all iconic SRD items are seeded and re-running migration causes no duplicate key violations.
- **AC3 (Dual Category & Type Filtering)**: Given catalog items, When querying items by category `POTION`, Then items with `item_category = 'POTION'` or `item_type ILIKE '%potion%'` or `item_type ILIKE '%poção%'` are returned.
- **AC4 (Common Rarity Normalization)**: Given items with rarity `COMMON`, `'NONE'`, or `NULL`, When querying items with rarity `COMMON`, Then all non-magical and common items are returned.
- **AC5 (Frontend Catalog Display)**: Given a user viewing `/encyclopedia/items` or opening `ItemCatalogModal`, When switching category to "Poções", "Itens Maravilhosos", or "Equipamento", Then populated item lists with accurate costs and details are displayed.

## Defensive Security & Validation Considerations
- **SQL Parameterization**: All repository queries use Spring Data parameterized queries (`:itemCategory`, `:rarity`, `:ruleset`), completely eliminating SQL injection risks.
- **Migration Safety**: Migration statements use `ON CONFLICT (slug) DO UPDATE` to prevent duplicate key errors and maintain idempotent execution.

## Testing Strategy & Shift-Left Plan
- **Migration Execution Test**: Flyway migration `V24` applied and verified against PostgreSQL.
- **Backend Repository & Service Tests**:
  - `ReferenceDataServiceTest` / `SpringDataItemRepository`:
    - Filter by `POTION` returns seeded potions.
    - Filter by `WONDROUS` returns seeded wondrous items.
    - Filter by `EQUIPMENT` returns adventuring gear.
    - Filter by `COMMON`, `UNCOMMON`, `RARE`, `VERY_RARE`, `LEGENDARY` returns corresponding items.
- **Frontend Unit Tests**:
  - `ItemCatalogModal` tests verify category switching displays items for Poções, Maravilhosos, and Equipamento.
  - `EncyclopediaItems` tests verify rarity filter selections.

## Risks and Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| Slug collisions during migration V24 seed | Low | All `INSERT` statements include `ON CONFLICT (slug) DO UPDATE` with canonical properties. |
| Query performance overhead on `ILIKE` fallback | Low | Item catalog is bounded (< 2,000 items); slug and category are indexed. |
| Frontend cache stale data | Low | TanStack Query invalidation keys include `['encyclopedia-items', category, rarity]`. |
