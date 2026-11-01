# Delta Spec: `player-character-domain`

> **Change ID:** `mechanical-properties-and-class-progression`
> **Target Canonical Spec:** `openspec/specs/player-character-domain.md`
> **Change Type:** Modify

## Architectural Updates

### 1. Polymorphic Item Mechanics Value Objects (`domain/model/`)
Zero framework imports allowed in `domain/model/`.
- **`ItemMechanics`** (sealed interface permitting `WeaponMechanics`, `ArmorMechanics`, `ConsumableMechanics`, `EmptyItemMechanics`):
  - **`WeaponMechanics`**:
    - `List<DamageRoll> damageRolls` (at least 1 required; each `DamageRoll` contains `String diceFormula` validated against `^\d{1,2}d(4|6|8|10|12|20|100)([+-]\d{1,3})?$` and `DamageType damageType`).
    - `String versatileDice` (nullable; validated against dice regex when `WeaponProperty.VERSATILE` is present).
    - `int magicBonus` (`0` to `3`).
    - `List<WeaponProperty> properties` (`FINESSE`, `LIGHT`, `HEAVY`, `REACH`, `TWO_HANDED`, `THROWN`, `AMMUNITION`, `LOADING`, `VERSATILE`).
  - **`ArmorMechanics`**:
    - `ArmorCategory armorCategory` (`LIGHT`, `MEDIUM`, `HEAVY`, `ROBE`, `SHIELD`).
    - `int baseAc` (`>= 1`).
    - `int magicBonus` (`0` to `3`).
    - `int strengthRequirement` (`>= 0`).
    - `boolean stealthDisadvantage`.
  - **`ConsumableMechanics`**:
    - `ConsumableEffectType effectType` (`HEAL_HP`, `GRANT_TEMP_HP`, `DEAL_DAMAGE`, `BUFF_UTILITY`).
    - `String diceFormula` (nullable for pure utility buffs; validated when present).
    - `String effectDescription`.

### 2. Unified Feat & Class Feature Value Object (`MechanicalEffects`)
- **`FeatType`**: `GENERAL`, `ORIGIN`, `CLASS`, `SUBCLASS`, `LINEAGE`.
- **`MechanicalEffects`**:
  - `Map<String, Integer> abilityBonuses` (e.g., `{"STR": 1}`).
  - `int acBonus`, `int initiativeBonus`, `int speedBonus`, `int hpPerLevelBonus`.
  - `List<String> grantedProficiencies`.
  - `String actionName`, `String actionType` (`ACTION`, `BONUS_ACTION`, `REACTION`, `SPECIAL`), `String diceFormula`.
  - `Integer usesPerRest`, `RestType restType` (`SHORT`, `LONG`, `NONE`).

### 3. `RulesEngine` Armor Class & Attack Calculation Rules
- **Dynamic Armor Class (`calculateArmorClass`)**:
  - **Body Slot Precedence**:
    - `HEAVY`: `baseAc + magicBonus` (ignores DEX modifier; disables Barbarian/Monk Unarmored Defense).
    - `MEDIUM`: `baseAc + magicBonus + Math.min(dexMod, 2)` (disables Unarmored Defense).
    - `LIGHT`: `baseAc + magicBonus + dexMod` (disables Unarmored Defense).
    - `ROBE` or **Unarmored**: Base starts at `robe.baseAc + robe.magicBonus + dexMod` (or `10 + dexMod` if no robe). If the character has Unarmored Defense (e.g., Barbarian `10 + dexMod + conMod` or Monk `10 + dexMod + wisMod`), the higher unarmored base formula applies, and `robe.magicBonus` (plus `robe.baseAc - 10` if `baseAc > 10`) stacks additively because `ROBE` does not count as armor.
  - **Shield & Passive Bonuses**:
    - Equipped `SHIELD` adds `shield.baseAc + shield.magicBonus` (typically `+2 + magicBonus`).
    - Passive `acBonus` from unlocked feats (`character_feats`) and equipped/attuned items stack additively.
- **Weapon Attack Calculation**:
  - Ability modifier: `STR` modifier by default; `Math.max(strMod, dexMod)` if `WeaponProperty.FINESSE` is present; `DEX` modifier for ranged weapons.
  - Attack bonus: `abilityMod + (isProficient ? proficiencyBonus : 0) + weapon.magicBonus`.
  - Primary damage formula: `damageRolls[0].diceFormula + (abilityMod + weapon.magicBonus)`, plus additional elemental/secondary `damageRolls[1..n]`.

### 4. Use Case 2 (UC2): XP Tracker, Level Up & Unrestricted Multiclassing
- **Official D&D 5e XP Table (Levels 1–20)**:
  - L1: `0`, L2: `300`, L3: `900`, L4: `2700`, L5: `6500`, L6: `14000`, L7: `23000`, L8: `34000`, L9: `48000`, L10: `64000`, L11: `85000`, L12: `100000`, L13: `120000`, L14: `140000`, L15: `165000`, L16: `195000`, L17: `225000`, L18: `265000`, L19: `305000`, L20: `355000`.
- **Add Experience (`POST /api/characters/{id}/xp`)**:
  - Rejects manual XP addition if `character.campaignId != null` (`422 Unprocessable Entity`).
  - Adds `amount > 0` to `character.experiencePoints` and returns updated `CharacterSheetResponse` including `canLevelUp: boolean` and `nextLevelXp: int`.
- **Level Up (`POST /api/characters/{id}/level-up`)**:
  - Supports **XP-based** (`experiencePoints >= nextLevelThreshold`) or **Milestone (`milestone = true`)**, which automatically sets `experiencePoints = Math.max(experiencePoints, thresholdForNewTotalLevel)`.
  - **Unrestricted Multiclassing**: Allows advancing an existing `ClassProgression` or adding any new `Archetype` without checking `abilityScore >= 13` prerequisites.
  - **Subclass Selection**: Required when the target class's new `classLevel == archetype.subclassLevel`.
  - **HP Increase**: `HpCalculationMethod` (`ROLL` referencing a valid server `HpRoll` token, `AVERAGE` using `(hitDie / 2) + 1`, or `MAX` using `hitDie`) + `conMod` (minimum 1 HP gained before feat bonuses) + sum of `hpPerLevelBonus` from character feats (e.g., *Tough* adds `+2` per level; if *Tough* is newly selected at this level, retroactively adds `2 * newTotalLevel`).
  - **Feature Unlock**: Automatically inserts all `archetype_feats` and `subclass_feats` matching `unlock_level == newClassLevel` into `character_feats` (initializing `currentCharges = usesPerRest`, `maxCharges = usesPerRest`).
  - **ASI / Feat Choice**: When `newClassLevel` is an ASI level for the class (`4, 8, 12, 16, 19`, plus `6, 14` for Fighter and `10` for Rogue), the request applies either an Ability Score Increase (`+2` to one ability or `+1/+1` to two abilities, each capped at `20`) OR adds a selected `GENERAL` (or `EPIC_BOON` at level 19+) Feat to `character_feats`.

### 5. Inventory, Consumables & Rest Endpoints
- `POST /api/characters/{id}/items`: Add catalog item (`itemId`, `quantity`).
- `PATCH /api/characters/{id}/items/{characterItemId}`: Toggle `equipped`, `attuned` (enforcing max 3 attuned items), or update `quantity`. Recalculates `armorClass` immediately.
- `DELETE /api/characters/{id}/items/{characterItemId}`: Remove item from inventory.
- `POST /api/characters/{id}/items/{characterItemId}/use`: Execute server-side dice roll for a `CONSUMABLE` item, apply `HEAL_HP` or `GRANT_TEMP_HP` to the character, decrement `quantity` by 1 (deleting the row if `quantity == 0`), and return `UseConsumableResponse` (`rollTotal`, `rollFormula`, `effectType`, `updatedCharacter`).
- `POST /api/characters/{id}/feats/{characterFeatId}/use`: Expend 1 charge (`currentCharges - 1`) and optionally roll the feature's `diceFormula`.
- `POST /api/characters/{id}/rest`: Execute `SHORT` rest (recharge `SHORT` rest features) or `LONG` rest (recharge `SHORT` and `LONG` rest features, restore `currentHp = maxHp`).
