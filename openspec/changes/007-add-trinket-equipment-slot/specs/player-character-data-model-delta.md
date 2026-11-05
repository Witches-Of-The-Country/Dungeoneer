# Delta Spec: `player-character-data-model`

> **Change ID:** `007-add-trinket-equipment-slot`
> **Target Canonical Spec:** `openspec/specs/player-character-data-model.md`
> **Change Type:** Modify

## Architectural Updates

### Equipment Slots Schema (`V31`)
- Migrates existing equipped items matching belt, girdle, charm, and trinket keywords (previously defaulted to `CHEST` under `V30`) to `equipped_slot = 'TRINKET'`.
- Database column `player_character.character_items.equipped_slot VARCHAR(30)` already supports `'TRINKET'` without schema change.
