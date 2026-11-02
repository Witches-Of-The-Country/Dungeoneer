# Player Character Data Model Delta: 008-srd-item-catalog-seed-and-category-normalization

## Database Migration `V24__normalize_srd_item_categories_and_seed_catalog.sql`

### 1. Normalization of `player_character.items.item_category`
Updates existing items in `player_character.items` where `item_category = 'MAGIC'`:
- Items with `item_type ILIKE '%poção%' OR item_type ILIKE '%potion%'` -> `item_category = 'POTION'`
- Items with `item_type ILIKE '%anel%' OR item_type ILIKE '%ring%'` -> `item_category = 'RING'`
- Items with `item_type ILIKE '%pergaminho%' OR item_type ILIKE '%scroll%'` -> `item_category = 'SCROLL'`
- Items with `item_type ILIKE '%varinha%' OR item_type ILIKE '%wand%'` -> `item_category = 'WAND'`
- Items with `item_type ILIKE '%bastão%' OR item_type ILIKE '%rod%'` -> `item_category = 'ROD'`
- Items with `item_type ILIKE '%cajado%' OR item_type ILIKE '%staff%'` -> `item_category = 'STAFF'`
- Items with `item_type ILIKE '%maravilhoso%' OR item_type ILIKE '%wondrous%'` -> `item_category = 'WONDROUS'`
- Items with `item_type ILIKE '%equipamento%' OR item_type ILIKE '%ferramenta%'` -> `item_category = 'EQUIPMENT'`

### 2. Seed Complete SRD Items
Populates:
- Complete SRD 2014 & SRD 2024 Potions (healing, buffs, utility, giant strength).
- Complete SRD Wondrous Items (protective, utility, movement, storage).
- Complete SRD Rings, Scrolls, Wands, Rods, Staves.
- Complete SRD Adventuring Gear, Packs, Tool Kits, and Focuses.

### 3. Repository Query Delta (`SpringDataItemRepository`)
Allows filtering with:
```sql
AND (CAST(:itemCategory AS VARCHAR) IS NULL
     OR UPPER(i.item_category) = UPPER(CAST(:itemCategory AS VARCHAR))
     OR UPPER(i.item_type) ILIKE CONCAT('%', UPPER(CAST(:itemCategory AS VARCHAR)), '%'))
AND (CAST(:rarity AS VARCHAR) IS NULL
     OR UPPER(i.rarity) = UPPER(CAST(:rarity AS VARCHAR))
     OR (UPPER(CAST(:rarity AS VARCHAR)) = 'COMMON' AND (i.rarity IS NULL OR UPPER(i.rarity) = 'NONE')))
```
