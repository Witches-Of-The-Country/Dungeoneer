# Tasks: Add Trinket Equipment Slot (12th Slot)

> **Change ID:** `007-add-trinket-equipment-slot`
> **Linked Proposal:** [proposal.md](./proposal.md)
> **Status:** completed
>
> All tasks must be `[x]` before this change can be archived via `sdd-spec-archive`.

---

## Legacy Data & Schema Migration Tasks

- [x] Create Flyway migration `V31__add_trinket_equipment_slot.sql` migrating existing equipped belts, girdles, talismans, charms, and trinkets to `equipped_slot = 'TRINKET'`.

## Implementation Tasks

- [x] Add `TRINKET` to `EquipmentSlot.java` in `player-character-service`.
- [x] Update `InventoryService.java` to infer `TRINKET` for belt/charm/trinket items and validate slot compatibility (rejecting armor/weapons/shields/rings in `TRINKET`).
- [x] Add `'TRINKET'` to `EquipmentSlot` union type in `dungeoneer-frontend/lib/api/characters.ts`.
- [x] Update `dungeoneer-frontend/components/character-sheet/inventory-tab.tsx` to include `TRINKET` (`Acessório / Cinto`) in `EQUIPMENT_SLOTS`, update header counters to 12 slots, and handle slot inference.

## Test Tasks

- [x] Add backend unit tests in `InventoryServiceTest.java` verifying `TRINKET` inference, auto-unequipping conflicting occupants, and invalid item rejection.
- [x] Update frontend unit tests in `inventory-tab-slots.test.tsx` verifying 12 slots rendering, slot counters, and `equipment-slot-trinket` interaction.

## Documentation Tasks

- [x] Update delta specs in `specs/player-character-domain-delta.md`, `specs/player-character-data-model-delta.md`, and `specs/frontend-web-app-delta.md`.

## Review Gate

- [x] All implementation tasks complete
- [x] All tests passing (`mvn clean test` and `npm test`)
- [x] Proposal acceptance criteria verified
- [x] Ready for `sdd-spec-archive`
