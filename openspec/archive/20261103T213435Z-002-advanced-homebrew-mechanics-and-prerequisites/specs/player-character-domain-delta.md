# Delta Spec: `player-character-domain`

> **Change ID:** `002-advanced-homebrew-mechanics-and-prerequisites`
> **Target Canonical Spec:** `openspec/specs/player-character-domain.md`
> **Change Type:** Modify

## Architectural Updates

### 1. Conditional Weapon Damage Rolls
- `DamageRoll` includes `boolean conditional` and `String conditionNote`.
- Weapons support an arbitrary list of `DamageRoll` entries (`1..N`).

### 2. Feat Choice Ability Bonuses, Conditional Effects & Hard Prerequisites
- `MechanicalEffects` supports `abilityBonusChoices` (`chooseCount`, `allowedAbilities`, `amount`), `conditional` boolean, and `prerequisiteRules` (`ABILITY_SCORE`, `FEAT`, `ARCHETYPE`, `LINEAGE`, `MIN_LEVEL`).
- `LevelUpService` and `CreateCharacterService` validate hard `prerequisiteRules` when a feat is selected.

### 3. Spell Mechanics, Class Spell Preparation & Lineage Feats
- `Spell` stores structured `mechanicalProperties` (damage rolls, effects, V/S/M components, casting time/range/duration units).
- `Archetype` includes `spellPreparationType` (`PREPARED` | `KNOWN` | `NONE`).
- `Lineage` supports linked/inline `lineage_feats` unlocked at character creation.
