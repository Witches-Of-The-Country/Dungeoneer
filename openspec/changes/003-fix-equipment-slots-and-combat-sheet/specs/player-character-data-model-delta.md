# Delta Spec: `player-character-data-model`

> **Change ID:** `003-fix-equipment-slots-and-combat-sheet`
> **Target Canonical Spec:** `openspec/specs/player-character-data-model.md`
> **Change Type:** Modify

## Architectural Updates

### Equipment Slots & Custom Actions Schema (`V30`)
- Adds `equipped_slot VARCHAR(30) NULL` to `player_character.character_items`.
- Migrates existing `is_equipped = TRUE` rows into their canonical slots and resolves duplicate equipped armors by keeping only the highest AC armor equipped in `CHEST`.
- Adds `player_character.character_custom_actions` table for persistent custom combat actions.
