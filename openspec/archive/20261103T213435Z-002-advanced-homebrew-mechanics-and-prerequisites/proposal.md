# Change Proposal: 002 — Advanced Homebrew Mechanics, Conditional Bonuses & Logical Prerequisite Builder

> **Change ID:** `002-advanced-homebrew-mechanics-and-prerequisites`
> **Status:** approved
> **Created:** 2026-11-01

---

## Summary

Upgrade the Homebrew Creation system and Domain Mechanics across `player-character-service` and `dungeoneer-frontend` to support:
1. **Weapons**: Incremental `+` button to add multiple `DamageRoll` entries (e.g. *Flame Tongue*: `1d8 SLASHING` + `2d6 FIRE`) with a `conditional` flag and condition label.
2. **Feats**:
   - Ability score bonuses supporting **choice pools** (`"Choose N among [STR, DEX, CON, ...]"`) in addition to fixed bonuses.
   - Logical **Prerequisite Builder** (`+` button) creating structured hard dependencies against database entities and character stats (`ABILITY_SCORE >= X`, `FEAT == featId`, `ARCHETYPE == archetypeId`, `LINEAGE == lineageId`, `MIN_LEVEL >= N`), enforced in character creation and level-up.
   - `conditional` flag on mechanical effect bonuses (`acBonus`, `savingThrowBonus`, `speedBonus`, etc.).
3. **Spells**:
   - Incremental `+` button to add mechanical damage rolls and structured spell effects.
   - Components as 3 checkboxes (`V`, `S`, `M` + optional material text).
   - Casting Time as numeric input + unit dropdown (`Ação`, `Ação Bônus`, `Reação`, `Minuto(s)`, `Hora(s)`).
   - Range as numeric input + unit dropdown (`Pés`, `Metros`, `Toque`, `Pessoal`, `Visão`).
   - Duration as `"Instantânea"` toggle OR numeric input + unit dropdown (`Rodada(s)`, `Minuto(s)`, `Hora(s)`, `Dia(s)`) + Concentration checkbox.
4. **Classes & Lineages**:
   - Class saving throw proficiencies as an interactive multi-select of the 6 ability scores.
   - Class spell preparation mode (`spellPreparationType`: `PREPARED`, `KNOWN`, `NONE`) alongside `spellcastingType`.
   - Inline feature/trait builder (`+ Adicionar Traço/Feat`) when creating a Homebrew Lineage.

---

## Motivation & Problem Statement

D&D 5e (2014 & 2024) items, feats, spells, classes, and lineages frequently involve multi-roll damage (e.g. *Flame Tongue* adding 2d6 conditional fire damage), choice-based ability score increases on half-feats (e.g. *Athlete*: +1 STR or DEX), hard prerequisites (e.g. Strength 13+, Elf lineage, Warlock class, or Level 4+), and structured spell casting parameters. Providing interactive builders instead of raw text fields ensures mechanical accuracy on the character sheet and prevents invalid data entry.

---

## Scope

### In Scope
- Expanding `DamageRoll` with `boolean conditional` and `String conditionNote` and supporting `1..N` damage rolls on Weapons and Spells.
- Expanding `MechanicalEffects` with `AbilityBonusChoice` (`chooseCount`, `allowedAbilities`, `bonusAmount`), `boolean conditional`, `String conditionDescription`, and structured `FeatPrerequisite` rules (`ABILITY_SCORE`, `FEAT`, `ARCHETYPE`, `LINEAGE`, `MIN_LEVEL`).
- Enforcing `FeatPrerequisite` rules in `CreateCharacterService` and `LevelUpService` and disabled/warning states in `LevelUpWizardModal`.
- Adding `spells.mechanical_properties JSONB`, `archetypes.spell_preparation_type VARCHAR(20)`, and `lineage_feats` join table via Flyway migration `V19`.
- Updating `CreateHomebrewDialog` in `dungeoneer-frontend` with incremental `+` builders for weapon damage, feat prerequisites, spell damage/effects, V/S/M checkboxes, casting time/range/duration unit dropdowns, class saving throw multi-select, class spell preparation mode, and inline lineage feats.

### Out of Scope
- Automated spell slot consumption during combat encounters (scheduled for Campaign Encounter tracker).
- Currency management and avatar uploads (covered in `004` and `005`).

---

## Proposed Solution

1. **Domain & Persistence Updates (`player-character-service`)**:
   - `DamageRoll`: add `boolean conditional` and `String conditionNote`.
   - `MechanicalEffects`: add `AbilityBonusChoice` (`int chooseCount`, `List<String> allowedAbilities`, `int bonusAmount`), `boolean conditional`, `String conditionDescription`, and `List<FeatPrerequisite> prerequisiteRules`.
   - `FeatPrerequisite`: structured rule record (`type`: `ABILITY_SCORE | FEAT | ARCHETYPE | LINEAGE | MIN_LEVEL`, `targetKey`: ability/UUID, `minValue`: int, `displayLabel`: string) validated in `CreateCharacterService` / `LevelUpService`.
   - `Spell`: add `mechanicalProperties` JSONB (`damageRolls`, `effects`, `concentration`, structured casting time/range/duration/components).
   - `Archetype`: add `spellPreparationType` (`PREPARED`, `KNOWN`, `NONE`).
   - `Lineage`: support inline `progressionFeats` creation in `CreateHomebrewLineageRequest` and `HomebrewService`.
2. **Frontend `CreateHomebrewDialog` & Character Sheet Updates (`dungeoneer-frontend`)**:
   - Weapon form: dynamic list of `damageRolls` with `+ Adicionar Dado de Dano`, `conditional` checkbox, and condition input.
   - Feat form: fixed vs `"Escolha N entre X, Y, Z"` ability bonus selector, `+ Adicionar Pré-requisito` builder (loading classes, lineages, and feats from the database), and `Condicional` checkbox on mechanical bonuses.
   - Spell form: `V / S / M` checkboxes, structured Casting Time / Range / Duration inputs with unit dropdowns + `"Instantânea"` option, and incremental `+ Adicionar Dado de Dano / Efeito` list.
   - Class form: 6-ability multi-select chips for Saving Throws and `spellPreparationType` selector (`Magias Preparadas`, `Magias Conhecidas`, `Nenhuma`).
   - Lineage form: inline `+ Adicionar Característica de Linhagem` builder.

---

## Impacted Files

| File Path | Layer | Reason |
|---|---|---|
| `player-character-service/.../db/migration/V19__add_spell_mechanics_archetype_prep_and_lineage_feats.sql` | Database | Add `spells.mechanical_properties`, `archetypes.spell_preparation_type`, and `lineage_feats`. |
| `player-character-service/.../domain/model/DamageRoll.java` | Domain | Add `conditional` flag and `conditionNote`. |
| `player-character-service/.../domain/model/MechanicalEffects.java` | Domain | Add `abilityBonusChoices`, `conditional` flag, and structured `prerequisiteRules`. |
| `player-character-service/.../domain/model/Spell.java` | Domain | Add structured `mechanicalProperties` (damage rolls, effects, V/S/M, units). |
| `player-character-service/.../domain/model/Archetype.java` | Domain | Add `spellPreparationType` (`PREPARED`, `KNOWN`, `NONE`). |
| `player-character-service/.../application/service/HomebrewService.java` | Application | Persist enhanced weapon/feat/spell/class/lineage mechanics and inline lineage feats. |
| `dungeoneer-frontend/components/homebrew/create-homebrew-dialog.tsx` | Frontend UI | Implement all incremental `+` builders, checkboxes, dropdowns, and multi-selects. |
| `dungeoneer-frontend/lib/api/homebrew.ts` | Frontend API | Update TypeScript payloads for weapons, feats, spells, classes, and lineages. |

---

## Acceptance Criteria

- **Given** a DM creating a Homebrew Weapon, **When** they click `+ Adicionar Dado de Dano`, **Then** they can add additional dice (e.g., `2d6 FIRE`) and mark it as `Condicional` (e.g., `"Enquanto a lâmina estiver acesa"`), and the character's `ActionsTab` displays both standard and conditional damage rolls.
- **Given** a DM creating a Homebrew Feat, **When** they configure `"Escolha 1 entre [FOR, DES] (+1)"` and add hard prerequisites (`Força >= 13`, `Classe == Guerreiro`), **Then** `LevelUpService` and `LevelUpWizardModal` enforce the hard prerequisite against the character's stats/class/lineage/feats and apply the chosen ability bonus.
- **Given** a DM creating a Homebrew Spell, **When** they select `V, S, M` checkboxes, configure numeric + unit dropdowns for Casting Time, Range, and Duration (or toggle `"Instantânea"`), and add incremental damage rolls/effects, **Then** `POST /api/reference/spells` persists both the formatted summary strings and the structured `mechanicalProperties` JSONB.
- **Given** a DM creating a Homebrew Class, **When** they select saving throws via the 6-ability multi-select chips and choose `Magias Preparadas` or `Magias Conhecidas`, **Then** `POST /api/reference/classes` persists `savingThrowProficiencies` and `spellPreparationType`.
- **Given** a DM creating a Homebrew Lineage, **When** they add inline features via `+ Adicionar Característica de Linhagem`, **Then** `POST /api/reference/lineages` creates the feats, links them in `lineage_feats`, and automatically unlocks them on characters created with that lineage.

---

## Defensive Security & Validation Considerations

- **Jakarta Validation & Bounds Checking**:
  - `AbilityBonusChoice`: validate `chooseCount >= 1 && chooseCount <= allowedAbilities.size()` and `bonusAmount` between `1` and `2`.
  - Dice formulas: validate against regex `^\d{1,2}d(4|6|8|10|12|20|100)([+-]\d{1,3})?$`.
  - Prerequisite graph safety: `FeatPrerequisite` evaluation checks only direct character state (`character.abilityScores`, `character.classProgressions`, `character.lineageId`, `character.feats`, `character.level`) in $O(1)$ without recursive graph traversal, preventing infinite loops or DoS.
- **Authorization**: All Homebrew creation endpoints (`POST /api/reference/spells`, `feats`, `items`, `classes`, `subclasses`, `lineages`) enforce `DUNGEON_MASTER` or `ADMIN` role via `RoleValidator`.

---

## Testing Strategy & Shift-Left Plan

| Scenario | Type | Target Test Class / File | Expected Outcome |
|---|---|---|---|
| Weapon with multiple damage rolls (standard + conditional) | Unit | `HomebrewServiceTest.java` | Persists all `DamageRoll` entries with `conditional = true` and `conditionNote`. |
| Feat with `AbilityBonusChoice` & hard `FeatPrerequisite` rules | Unit | `LevelUpServiceTest.java` | Rejects level-up feat selection when hard prerequisite (`ABILITY_SCORE`, `ARCHETYPE`, `LINEAGE`, `FEAT`) is unmet; applies chosen ability bonus when met. |
| Spell with V/S/M, unit dropdowns, and damage rolls | Unit | `HomebrewServiceTest.java` | Persists `Spell` with `mechanicalProperties` JSONB. |
| Lineage with inline `progressionFeats` | Unit | `HomebrewServiceTest.java`, `CreateCharacterServiceTest.java` | Creates lineage feats and unlocks them on new characters of that lineage. |
| Frontend `CreateHomebrewDialog` incremental builders | Unit | `__tests__/homebrew-creation.test.tsx` | Verifies `+` buttons for weapon damage, feat prerequisites, spell mechanics, class multi-select saving throws, and lineage inline feats. |

- **Verification Commands**:
  - `mvn -f player-character-service/backend/pom.xml test`
  - `npm --prefix dungeoneer-frontend run test:unit`

---

## Risks and Mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| Backwards compatibility with existing `DamageRoll` and `MechanicalEffects` JSONB rows | Medium | Default missing `conditional` fields to `false` and missing `prerequisiteRules` / `abilityBonusChoices` to empty lists during JSONB deserialization. |
| Complex form state in `CreateHomebrewDialog` | Low | Encapsulate each sub-builder (Damage Roll list, Prerequisite list, Progression Feat list) with clean array state helpers and sensible defaults. |
