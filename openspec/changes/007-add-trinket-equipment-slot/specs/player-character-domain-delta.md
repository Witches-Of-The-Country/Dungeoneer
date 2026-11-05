# Delta Spec: `player-character-domain`

> **Change ID:** `007-add-trinket-equipment-slot`
> **Target Canonical Spec:** `openspec/specs/player-character-domain.md`
> **Change Type:** Modify

## Architectural Updates

### Equipment Slots System (`EquipmentSlot`) Expansion
- Supported slots expands from 11 to 12 slots: `CHEST` (peito), `GLOVES` (luvas), `BOOTS` (botas), `HELMET` (capacete), `PANTS` (calças), `LEFT_HAND` (mão esquerda), `RIGHT_HAND` (mão direita), `CLOAK` (capa), `RING_1` (anel 1), `RING_2` (anel 2), `NECKLACE` (colar), and `TRINKET` (acessório / cinto).
- Slot invariants enforced by `InventoryService`:
  - `TRINKET` holds belts, girdles, sashes, talismans, charms, and trinkets.
  - Equipping an item into `TRINKET` when another item occupies `TRINKET` automatically unequips the previous item.
  - Armors, weapons, shields, and rings cannot be equipped into `TRINKET` (`CharacterValidationException`).
