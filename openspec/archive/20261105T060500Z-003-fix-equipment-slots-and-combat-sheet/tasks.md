# Tasks: Equipment Slots System, Temp HP Damage Absorption & Custom Actions CRUD

> **Change ID:** `003-fix-equipment-slots-and-combat-sheet`
> **Execution Order:** `003`
> **Linked Proposal:** [proposal.md](./proposal.md)
> **Status:** completed
>
> All tasks must be `[x]` before this change can be archived via `sdd-spec-archive`.

---

## Legacy Data & Schema Migration Tasks

- [x] Create Flyway migration `V30__add_equipment_slots_and_custom_actions.sql` adding `equipped_slot VARCHAR(30)` to `player_character.character_items` and migrating existing equipped items (`CHEST` for single highest-AC armor, unequipping duplicate armors; `LEFT_HAND` for shields; `RIGHT_HAND` for weapons; `RING_1`/`RING_2` for rings; accessory slots for wondrous items).
- [x] Add `player_character.character_custom_actions` table in `V30` and support transparent migration of existing `localStorage` custom actions in `actions-tab.tsx`.

## Implementation Tasks

- [x] Create `EquipmentSlot.java` enum (`CHEST`, `GLOVES`, `BOOTS`, `HELMET`, `PANTS`, `LEFT_HAND`, `RIGHT_HAND`, `CLOAK`, `RING_1`, `RING_2`, `NECKLACE`) and update `CharacterItem`, `CharacterItemJpaEntity`, and `UpdateCharacterItemCommand`.
- [x] Update `InventoryService.java` and `CharacterSheetHydrator.java` to enforce slot occupancy rules (auto-unequip previous occupant of the same slot, lock `LEFT_HAND` when `TWO_HANDED` weapon is equipped in `RIGHT_HAND`, limit rings to `RING_1`/`RING_2`).
- [x] Add `takeDamage(int amount)` to `PlayerCharacter.java` and update `app/characters/[id]/page.tsx` (`handleApplyDamage` and `handleAdjustHp`) so incoming damage deducts from `hitPointsTemp` first before reducing `hitPointsCurrent`.
- [x] Implement Edit and Delete operations for custom actions in `dungeoneer-frontend/components/character-sheet/actions-tab.tsx` (and backend endpoints if applicable).
- [x] Update `dungeoneer-frontend/components/character-sheet/inventory-tab.tsx` to display the 11 equipment slots and handle slot-based equipping.

## Test Tasks

- [x] Add unit/integration tests in `InventoryServiceTest` verifying slot replacement (cannot have 2 armors in `CHEST`, 2-handed weapon unequips shield, max 2 rings).
- [x] Add unit tests in `PlayerCharacterTest` and frontend tests verifying Temp HP absorbs partial and overflow damage accurately.
- [x] Add frontend unit tests in `ActionsTab` verifying creating, editing, and deleting a custom action.

## Documentation Tasks

- [x] Update delta specs in `specs/player-character-domain-delta.md`, `specs/player-character-data-model-delta.md`, and `specs/frontend-web-app-delta.md`.

## Review Gate

- [x] All implementation tasks complete
- [x] All tests passing (`mvn clean test` and `npm test`)
- [x] Proposal acceptance criteria verified
- [x] Ready for `sdd-spec-archive`
