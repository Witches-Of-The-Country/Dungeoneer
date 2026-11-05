# Proposal: Add Trinket Equipment Slot (12th Slot)

> **Change ID:** `007-add-trinket-equipment-slot`
> **Status:** completed
> **Author:** @jaca
> **Created:** 2026-11-05
> **Last Updated:** 2026-11-05

---

## Summary

Adds a 12th canonical equipment slot: **`TRINKET`** (for belts, girdles, charms, talismans, and miscellaneous wondrous items) to the Dungeoneer equipment system across domain models, Flyway migrations, inventory services, UI components, and test suites.

## Motivation

In D&D 5e/2024, characters frequently equip belts (such as *Belt of Giant Strength*, *Belt of Dwarvenkind*), sashes, talismans, charms, and trinkets. With the previous 11-slot system (`CHEST`, `GLOVES`, `BOOTS`, `HELMET`, `PANTS`, `LEFT_HAND`, `RIGHT_HAND`, `CLOAK`, `RING_1`, `RING_2`, `NECKLACE`), belts and charms had no designated slot and fell back to `CHEST` or generic wondrous items. Adding the discrete `TRINKET` slot provides dedicated visual and mechanical placement, prevents belt/armor conflicts, and allows seamless attunement and stat hydration.

## Scope

### In Scope
- Add `TRINKET` to `EquipmentSlot` domain enum.
- Update `InventoryService.resolveSlotForItem` with keyword matching for belts, sashes, talismans, charms, and trinkets (`cinto`, `cinturão`, `cinturao`, `faixa`, `belt`, `girdle`, `charm`, `talismã`, `talisma`, `talisman`, `trinket`, `berloque`).
- Update `InventoryService.validateSlotCompatibility` to reject body armor, weapons, shields, and rings in `TRINKET`.
- Create Flyway migration `V31__add_trinket_equipment_slot.sql` migrating existing equipped belts/charms from `CHEST` to `TRINKET`.
- Update frontend `lib/api/characters.ts` union type `EquipmentSlot` with `'TRINKET'`.
- Update `dungeoneer-frontend/components/character-sheet/inventory-tab.tsx` to render the 12th slot (`Acessório / Cinto`), update header counters ("12 Slots", "X / 12 ocupados"), and support default inference and slot removal.
- Add backend unit tests in `InventoryServiceTest` and frontend tests in `inventory-tab-slots.test.tsx`.

### Out of Scope
- Visual 3D paper-doll avatar rendering.
- Multiple trinket slots (standard D&D limits to one active belt/trinket slot alongside necklace and rings).

## Proposed Solution

### Approach
1. **Domain Model**: Expand `EquipmentSlot` in `player-character-service` with `TRINKET`. `CharacterItem`, `CharacterItemJpaEntity` (`VARCHAR(30)`), and DTOs seamlessly support this new value.
2. **Inventory Rules**: Update `InventoryService` to infer `TRINKET` for belt/charm/trinket items, enforce single-slot replacement for `TRINKET`, and prohibit equipping non-trinket items (like body armor or weapons) in `TRINKET`.
3. **Database Migration**: Create `V31__add_trinket_equipment_slot.sql` updating any equipped character item whose name matches belt or charm keywords to `equipped_slot = 'TRINKET'`.
4. **Frontend UI**: Add `'TRINKET'` to TypeScript types and the `EQUIPMENT_SLOTS` list in `InventoryTab`. The UI will display 12 slots arranged in a responsive grid, with the label "Acessório / Cinto" and test ID `equipment-slot-trinket`.

### Impacted Files
| File | Change Type | Notes |
|------|-------------|-------|
| `player-character-service/.../domain/model/EquipmentSlot.java` | Modify | Add `TRINKET` enum member |
| `player-character-service/.../application/service/InventoryService.java` | Modify | Slot inference and compatibility validation for `TRINKET` |
| `player-character-service/.../db/migration/V31__add_trinket_equipment_slot.sql` | Create | Migrate equipped belts/charms to `TRINKET` |
| `player-character-service/.../application/service/InventoryServiceTest.java` | Modify | Add unit tests for `TRINKET` slot inference, replacement, and validation |
| `dungeoneer-frontend/lib/api/characters.ts` | Modify | Add `'TRINKET'` to `EquipmentSlot` type |
| `dungeoneer-frontend/components/character-sheet/inventory-tab.tsx` | Modify | Add 12th slot card, counter, and label mapper |
| `dungeoneer-frontend/__tests__/inventory-tab-slots.test.tsx` | Modify | Verify 12 slots rendering, slot counters, and `TRINKET` interaction |

### Data Model Changes
- Flyway migration `V31` updates legacy rows in `player_character.character_items`:
  ```sql
  UPDATE player_character.character_items ci
  SET equipped_slot = 'TRINKET'
  FROM player_character.items i
  WHERE ci.item_id = i.id
    AND ci.is_equipped = TRUE
    AND (ci.equipped_slot = 'CHEST' OR ci.equipped_slot IS NULL)
    AND (
      i.name ILIKE '%cinto%' OR i.name ILIKE '%cinturão%' OR i.name ILIKE '%cinturao%' OR
      i.name ILIKE '%belt%' OR i.name ILIKE '%girdle%' OR i.name ILIKE '%charm%' OR
      i.name ILIKE '%talismã%' OR i.name ILIKE '%talisma%' OR i.name ILIKE '%talisman%' OR
      i.name ILIKE '%trinket%' OR i.name ILIKE '%faixa%'
    );
  ```

## Acceptance Criteria

- [x] **Given** a character equipping a belt or charm item (e.g. *Belt of Giant Strength* or *Cinto de Força*) without specifying a slot **When** `InventoryService.updateItem` is called with `equipped = true` **Then** the item is assigned to `EquipmentSlot.TRINKET`.
- [x] **Given** a character with an item already equipped in `TRINKET` **When** they equip a second trinket/belt in `TRINKET` **Then** the previous item in `TRINKET` is automatically unequipped.
- [x] **Given** a character attempting to equip a body armor or weapon into `TRINKET` **When** `validateSlotCompatibility` runs **Then** a `CharacterValidationException` is thrown.
- [x] **Given** a user viewing their character's inventory tab in the frontend **When** the page renders **Then** 12 equipment slots are displayed, including `Acessório / Cinto` (`equipment-slot-trinket`), and the header indicates `"Slots de Equipamento (12 Slots)"`.
- [x] **Given** existing database records with belts equipped under `CHEST` prior to `V31` **When** the migration runs **Then** their `equipped_slot` is updated to `'TRINKET'`.

## Defensive Security & Validation Considerations

### Input Validation
- Inbound `equippedSlot` strings validated by Jackson enum deserialization against `EquipmentSlot`.
- Server-side slot compatibility checks enforce that armors and weapons cannot be injected into `TRINKET`.

### Secret Isolation
- N/A.

### Authorization & Authentication Impact
- Inventory mutations remain scoped to character ownership via `X-User-Id`.

### Other Security Risks
- Prevents AC calculation manipulation from slot tampering.

## Testing Strategy & Shift-Left Plan

### Happy Path Tests
| Test ID | Scenario | Test Type | Expected Result |
|---------|----------|-----------|-----------------|
| TP-01 | Infer and equip belt item in `TRINKET` | Unit (`InventoryServiceTest`) | `charItem.getEquippedSlot() == EquipmentSlot.TRINKET` |
| TP-02 | Equip second trinket item | Unit (`InventoryServiceTest`) | Previous trinket unequipped, new trinket in `TRINKET` |
| TP-03 | Render 12 equipment slots in InventoryTab | Frontend Unit (`inventory-tab-slots.test.tsx`) | 12 slots visible, `equipment-slot-trinket` present |

### Edge Case & Failure Mode Tests
| Test ID | Scenario | Test Type | Expected Result |
|---------|----------|-----------|-----------------|
| TE-01 | Attempt to equip Armor in `TRINKET` | Unit (`InventoryServiceTest`) | Throws `CharacterValidationException` |
| TE-02 | Unequip item currently in `TRINKET` from slot grid | Frontend Unit (`inventory-tab-slots.test.tsx`) | Triggers mutation with `equipped = false, equippedSlot = null` |

### Test Commands
- **Backend tests:** `mvn clean test`
- **Frontend tests:** `npm test`

## Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Unmatched Portuguese/English trinket names | Low | Low | Comprehensive keyword matching covering synonyms (`cinto`, `cinturão`, `belt`, `girdle`, `charm`, `talismã`, `trinket`, `faixa`) with user override support in UI |

## Specification Changes & Delta Stubs

### `specs/player-character-domain-delta.md`
- **Target Canonical Spec:** `openspec/specs/player-character-domain.md`
- **Summary of Updates:** Document the 12th `TRINKET` slot in `EquipmentSlot` and slot resolution rules.

### `specs/player-character-data-model-delta.md`
- **Target Canonical Spec:** `openspec/specs/player-character-data-model.md`
- **Summary of Updates:** Document Flyway migration `V31__add_trinket_equipment_slot.sql`.

### `specs/frontend-web-app-delta.md`
- **Target Canonical Spec:** `openspec/specs/frontend-web-app.md`
- **Summary of Updates:** Document 12-slot inventory grid and `TRINKET` card in `InventoryTab`.
