# Delta Spec: `infrastructure-and-testing`

> **Change ID:** `mechanical-properties-and-class-progression`
> **Target Canonical Spec:** `openspec/specs/infrastructure-and-testing.md`
> **Change Type:** Modify

## Architectural Updates

### 1. Backend Shift-Left Testing Strategy (`player-character-service`)
- **Domain Unit Tests (JUnit 5, zero Spring context)**:
  - `ItemMechanicsTest`: Validates invariants on `WeaponMechanics` (composite damage rolls, dice regex, versatile die), `ArmorMechanics` (`LIGHT`, `MEDIUM`, `HEAVY`, `ROBE`, `SHIELD`), and `ConsumableMechanics`.
  - `Srd2014RulesEngineTest` & `Srd2024RulesEngineTest`: Verifies dynamic AC calculation across all 5 `ArmorCategory` values, Unarmored Defense interaction with `ROBE` vs standard armor, passive `acBonus` stacking, and XP threshold table resolution.
  - `LevelUpServiceTest`: Verifies XP vs Milestone level-up, unrestricted multiclassing, Subclass binding at `subclassLevel`, HP calculation (`ROLL`, `AVERAGE`, `MAX` + CON mod + `hpPerLevelBonus`), automatic unlocking of `CLASS` and `SUBCLASS` feats, and ASI (+2 or +1/+1, max 20) vs Feat branching.
  - `InventoryServiceTest` & `RestServiceTest`: Verifies equip/unequip AC recalculation, max 3 attuned items enforcement, consumable dice rolling with auto-decrement/removal at `quantity == 0`, and Short/Long Rest charge recovery.
- **Testcontainers Integration Tests (`PostgreSQL 15`)**:
  - Verifies clean execution of Flyway migrations `V16`, `V17`, and `V18` on top of `V1`–`V15`.
  - Verifies JSONB round-trip persistence for `items.mechanical_properties` and `feats.mechanical_effects`.
  - Verifies end-to-end MockMvc flows for `POST /api/characters/{id}/xp`, `POST /api/characters/{id}/level-up`, `POST /api/characters/{id}/rest`, `POST/PATCH/DELETE /api/characters/{id}/items`, `POST /api/characters/{id}/items/{characterItemId}/use`, and `POST /api/reference/homebrew/subclasses`.
- **Coverage Gate**:
  - `mvn clean verify` enforces JaCoCo `>= 80%` line coverage across `domain` and `application` packages.

### 2. Frontend Testing Strategy (`dungeoneer-frontend`)
- **Vitest + React Testing Library**:
  - `character-sheet.test.tsx`: Verifies `Inventário` tab (adding items, equipping armor/robes/shields, using healing potions), `Talentos` tab (grouped features, charge expenditure, Short/Long Rest), and `Ações` tab (weapon attacks, versatile rolls, dice roller modal).
  - `level-up-wizard.test.tsx`: Verifies `XpProgressBar`, `AddXpModal` (including campaign lock when `campaignId != null`), and `LevelUpWizardModal` (multiclassing, subclass selection, HP roll/average, ASI vs Feat).
  - `homebrew-creation.test.tsx` & `encyclopedia-browse.test.tsx`: Verifies dynamic mechanical fields in `CreateHomebrewDialog` and the 1–20 class progression table in `/encyclopedia/classes`.
