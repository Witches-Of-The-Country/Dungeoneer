# Bugfix Proposal: Equipment Slots System, Temp HP Damage Absorption & Custom Actions CRUD

> **Change ID:** `003-fix-equipment-slots-and-combat-sheet`
> **Execution Order:** `003`
> **Type:** Bugfix & Legacy Adaptation
> **Priority:** High
> **Status:** completed
> **Created:** 2026-11-04
> **Last Updated:** 2026-11-04

---

## Summary

Addresses three core Character Sheet & Inventory issues:
1. Replaces the unconstrained boolean `equipped` toggle with a D&D 5e/2024 **Equipment Slots System** (`CHEST`/peito, `GLOVES`/luvas, `BOOTS`/botas, `HELMET`/capacete, `PANTS`/calças, `LEFT_HAND`/mão esquerda, `RIGHT_HAND`/mão direita, `CLOAK`/capa, `RING_1` & `RING_2`/anéis [max 2], `NECKLACE`/colar), preventing characters from equipping 2 body armors or equipping a shield alongside a two-handed weapon.
2. Fixes damage application so **Temporary Hit Points (`hitPointsTemp`) absorb incoming damage first** before reducing true HP (`hitPointsCurrent`).
3. Implements **Edit and Delete operations for Custom Actions** ("Ações Personalizadas") in the Actions tab, backed by persistent storage and seamless migration of existing client-side custom actions.

## Grouped Bugs & Symptoms

- **Bug 1:** Equipar 2 armaduras é permitido — vamos criar slots (peito, luvas, botas, capacete, calças, mão esquerda, mão direita, capa, anéis [consultar máx do D&D = 2 anéis], e colar).
- **Bug 2:** Quando toma dano e tiver PV temporário, não sai dos PVs temporários, sai do HP verdadeiro.
- **Bug 3:** Editar/excluir ações personalizadas — não tem nenhuma das duas operações implementadas.

## Root Cause Analysis (Causa Raiz)

1. **Multiple Armors Equipped (`CharacterItem` & `InventoryService`)**:
   - `CharacterItem.java` and `player_character.character_items` only store `is_equipped BOOLEAN`.
   - `InventoryService.updateItem` toggles `equipped` without checking item category or slot occupancy.
   - `CharacterSheetHydrator` loops through all equipped items and overwrites `equippedArmor` with whichever armor came last, leaving multiple armors marked `equipped = true` in the UI.
2. **Temporary HP Bypassed on Damage (`app/characters/[id]/page.tsx`)**:
   - In `dungeoneer-frontend/app/characters/[id]/page.tsx` (lines 181–209), `handleApplyDamage` calls `handleAdjustHp(-amount)`, which directly subtracts from `character.hitPointsCurrent` and sends `hitPointsTemp: character.hitPointsTemp` untouched in the `PUT` payload.
   - Furthermore, the damage controls are disabled when `character.hitPointsCurrent <= 0` even if `character.hitPointsTemp > 0`.
3. **Missing Edit/Delete for Custom Actions (`actions-tab.tsx`)**:
   - In `dungeoneer-frontend/components/character-sheet/actions-tab.tsx` (lines 74–130), custom actions (`category === 'CUSTOM'`) only implement creation (`handleCreateCustomAction`). No Edit modal, Delete handler, or action buttons exist on custom action cards, and custom actions are only stored in `localStorage`.

## Systemic Ripple Effects & Similar Latent Bugs (Repercussão no Código)

- **Two-Handed Weapons vs Shields**: Equipping a weapon with `WeaponProperty.TWO_HANDED` must occupy both `RIGHT_HAND` and `LEFT_HAND` (auto-unequipping or blocking any shield/off-hand weapon in `LEFT_HAND`). Conversely, equipping a shield in `LEFT_HAND` while wielding a two-handed weapon must unequip the two-handed weapon.
- **Ring Limit in D&D 5e/2024**: Standard D&D rules limit characters to wearing at most **2 magic rings** (one on each hand: `RING_1` and `RING_2`).
- **Healing vs Damage Separation**: Healing (`handleApplyHeal`) must continue restoring only `hitPointsCurrent` (up to `hitPointsMax`) and never add to `hitPointsTemp`, whereas Damage (`handleApplyDamage` and `-1 PV`) must deduct from `hitPointsTemp` first and spill over any remainder to `hitPointsCurrent`.

## Legacy Code & Data Adaptation Strategy (Adequação de Legado)

### Legacy Impact Matrix
| Legacy Artifact / Table | Discrepancy Identified | Adaptation / Migration Action |
|-------------------------|------------------------|-------------------------------|
| `player_character.character_items` | Existing equipped items have `is_equipped = true` but no `equipped_slot` | Flyway migration `V30__add_equipment_slots_and_custom_actions.sql` adds `equipped_slot VARCHAR(30) NULL` and migrates existing equipped items: body armors → `CHEST` (if a character has 2+ armors equipped, only the highest AC armor stays equipped in `CHEST`, others set `is_equipped = false`), shields → `LEFT_HAND`, weapons → `RIGHT_HAND`, rings → `RING_1`/`RING_2`, boots/gloves/helmets/cloaks/necklaces → respective slots. |
| `localStorage` (`dungeoneer_actions_<characterId>`) | Beta testers created custom actions in browser `localStorage` | `actions-tab.tsx` implements full Edit and Delete for custom actions (syncing with backend persistence and/or `localStorage` fallback so zero existing custom actions are lost). |

## Scope

### In Scope
- Define `EquipmentSlot` enum (`CHEST`, `GLOVES`, `BOOTS`, `HELMET`, `PANTS`, `LEFT_HAND`, `RIGHT_HAND`, `CLOAK`, `RING_1`, `RING_2`, `NECKLACE`).
- Add `equipped_slot` to `player_character.character_items` via Flyway migration `V30` and migrate legacy equipped items.
- Enforce slot rules in `InventoryService` (1 item per slot; auto-unequip or reject conflicting slot; `TWO_HANDED` weapons lock both hands; max 2 rings across `RING_1`/`RING_2`).
- Update `inventory-tab.tsx` to display the character's equipped slots (peito, luvas, botas, capacete, calças, mão esquerda, mão direita, capa, anel 1, anel 2, colar) and slot selection on equip.
- Implement D&D 5e Temp HP absorption in `PlayerCharacter.takeDamage(int)` and frontend `handleApplyDamage` / `-1 PV` button.
- Implement Edit and Delete operations (UI buttons, edit modal, confirmation, and backend/storage persistence) for Custom Actions in `actions-tab.tsx`.

### Out of Scope
- Visual 3D paper-doll avatar rendering.

## Proposed Solution

### Impacted Files
| File | Change Type | Notes |
|------|-------------|-------|
| `player-character-service/backend/src/main/resources/db/migration/V30__add_equipment_slots_and_custom_actions.sql` | Create | Add `equipped_slot` column to `character_items`, migrate legacy equipped items, and create `character_custom_actions` table |
| `player-character-service/.../domain/model/EquipmentSlot.java` | Create | Enum with the 11 equipment slots (`CHEST`, `GLOVES`, `BOOTS`, `HELMET`, `PANTS`, `LEFT_HAND`, `RIGHT_HAND`, `CLOAK`, `RING_1`, `RING_2`, `NECKLACE`) |
| `player-character-service/.../domain/model/CharacterItem.java` | Modify | Add `EquipmentSlot equippedSlot` field |
| `player-character-service/.../domain/model/PlayerCharacter.java` | Modify | Add `takeDamage(int amount)` domain logic absorbing `hitPointsTemp` before `hitPointsCurrent` |
| `player-character-service/.../application/service/InventoryService.java` | Modify | Validate and resolve slot assignment, single-armor rule, 2-ring rule, and `TWO_HANDED` hand locking |
| `player-character-service/.../application/service/CharacterSheetHydrator.java` | Modify | Resolve AC from `CHEST` armor and `LEFT_HAND`/`RIGHT_HAND` shield |
| `dungeoneer-frontend/components/character-sheet/inventory-tab.tsx` | Modify | Render Equipment Slots overview and slot picker when equipping items |
| `dungeoneer-frontend/app/characters/[id]/page.tsx` | Modify | Deduct damage from `hitPointsTemp` first, spilling remainder to `hitPointsCurrent` |
| `dungeoneer-frontend/components/character-sheet/actions-tab.tsx` | Modify | Add Edit & Delete buttons and modal for `CUSTOM` actions |

## Acceptance Criteria

- [x] **Given** a character with an armor equipped in `CHEST` **When** they equip a second armor **Then** the previous armor in `CHEST` is automatically unequipped (or replaced) so two armors are never equipped simultaneously.
- [x] **Given** a character equipping rings **When** `RING_1` and `RING_2` are occupied **Then** equipping a third ring requires replacing `RING_1` or `RING_2`.
- [x] **Given** a character with `10` Current HP and `5` Temporary HP **When** they take `3` damage **Then** Temporary HP becomes `2` and Current HP remains `10`.
- [x] **Given** a character with `10` Current HP and `5` Temporary HP **When** they take `8` damage **Then** Temporary HP becomes `0` and Current HP becomes `7`.
- [x] **Given** a character with a Custom Action in the Actions tab **When** the user clicks Edit or Delete **Then** they can modify its name, action type, attack bonus, damage formula/type, and description, or delete it permanently.
- [x] **Given** an existing character in the database with 2 armors marked `is_equipped = true` prior to migration `V30` **When** the migration runs **Then** only one armor remains equipped in `CHEST`.

## Defensive Security & Validation Considerations

### Input Validation
- `EquipmentSlot` validated against the allowed enum values and item compatibility.
- Custom action fields validated for length, dice formula syntax, and non-blank name.
- Damage and heal amounts validated as positive integers (`>= 0`).

### Secret Isolation
- N/A.

### Authorization & Authentication Impact
- Inventory, HP, and custom action mutations require character ownership (`X-User-Id`).

### Other Security Risks
- Server-side enforcement of single-slot occupancy prevents client payload tampering to stack multiple armors/shields.

## Testing Strategy & Shift-Left Plan

### Regression & Happy Path Tests
| Test ID | Scenario | Test Type | Expected Result |
|---------|----------|-----------|-----------------|
| TR-01 | Equip second armor when `CHEST` already has armor | Unit / Integration | Previous armor is unequipped (`equipped = false, slot = null`), new armor occupies `CHEST` |
| TR-02 | Apply 6 damage to character with 12 HP and 4 Temp HP | Unit (Backend & Frontend) | Temp HP = 0, Current HP = 10 |
| TR-03 | Edit and Delete a custom action in `ActionsTab` | Frontend Unit (Vitest) | Updated fields render immediately; deleted action is removed |

### Edge Case & Failure Mode Tests
| Test ID | Scenario | Test Type | Expected Result |
|---------|----------|-----------|-----------------|
| TE-01 | Equip a `TWO_HANDED` weapon while a Shield is in `LEFT_HAND` | Unit | Shield in `LEFT_HAND` is automatically unequipped |
| TE-02 | Take damage when `hitPointsCurrent == 0` but `hitPointsTemp == 5` | Unit | Temp HP absorbs the damage |

### Test Commands
- **Backend tests:** `mvn clean test`
- **Frontend tests:** `npm test`

## Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Legacy items without explicit slot metadata | Med | Low | Default slot inference by `itemCategory` / `itemType` / keywords, with fallback allowing user to choose among valid accessory/hand slots |

## Specification Changes & Delta Stubs

### `specs/player-character-domain-delta.md`
- **Target Canonical Spec:** `openspec/specs/player-character-domain.md`
- **Summary of Updates:** Document the 11-slot `EquipmentSlot` model, two-handed weapon locking, D&D 5e Temp HP absorption order, and Custom Actions CRUD.

### `specs/player-character-data-model-delta.md`
- **Target Canonical Spec:** `openspec/specs/player-character-data-model.md`
- **Summary of Updates:** Document `equipped_slot` column on `character_items` and `V30` legacy migration rules.

### `specs/frontend-web-app-delta.md`
- **Target Canonical Spec:** `openspec/specs/frontend-web-app.md`
- **Summary of Updates:** Document Equipment Slots UI in `InventoryTab`, Temp HP damage deduction in Character Sheet, and Custom Action Edit/Delete modals in `ActionsTab`.
