# Delta Spec: `player-character-domain`

> **Change ID:** `003-fix-equipment-slots-and-combat-sheet`
> **Target Canonical Spec:** `openspec/specs/player-character-domain.md`
> **Change Type:** Modify

## Architectural Updates

### Equipment Slots System (`EquipmentSlot`)
- Supported slots: `CHEST` (peito), `GLOVES` (luvas), `BOOTS` (botas), `HELMET` (capacete), `PANTS` (calças), `LEFT_HAND` (mão esquerda), `RIGHT_HAND` (mão direita), `CLOAK` (capa), `RING_1` (anel 1), `RING_2` (anel 2), `NECKLACE` (colar).
- Slot invariants enforced by `InventoryService`:
  - Each slot holds at most 1 item. Equipping a new item into an occupied slot unequips the previous item in that slot.
  - Armors can only be equipped in `CHEST`.
  - Shields can only be equipped in `LEFT_HAND` (or `RIGHT_HAND`).
  - Weapons with `WeaponProperty.TWO_HANDED` occupy `RIGHT_HAND` and lock `LEFT_HAND` (unequipping any item in `LEFT_HAND`).
  - Rings occupy `RING_1` or `RING_2` (maximum 2 rings equipped simultaneously).

### Temporary Hit Points Damage Absorption
- When a character takes `damage > 0`:
  1. `absorbedByTemp = Math.min(hitPointsTemp, damage)`
  2. `newTempHp = hitPointsTemp - absorbedByTemp`
  3. `remainingDamage = damage - absorbedByTemp`
  4. `newCurrentHp = Math.max(0, hitPointsCurrent - remainingDamage)`
