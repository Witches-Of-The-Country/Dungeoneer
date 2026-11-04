# Delta Spec: `player-character-domain`

> **Change ID:** `add-homebrew-feat-charges`  
> **Target Canonical Spec:** `openspec/specs/player-character-domain.md`  
> **Change Type:** Modify

---

## 1. Architectural Updates

### 1.1 Enums & Value Objects
- **`ChargeType`** (`com.dungeoneer.playerCharacter.domain.model.ChargeType`):
  - `FIXED`: Static number of charges declared in `fixedMaxCharges`.
  - `ABILITY_MODIFIER`: Derived from the character's modifier in `chargeAbility` (STR, DEX, CON, INT, WIS, CHA), clamped to a minimum of 1.
  - `PROFICIENCY`: Derived from character's proficiency bonus (calculated as `2 + (level - 1) / 4`).
  - `LEVEL`: Derived from total character level.
- **`ChargeRecoveryType`** (`com.dungeoneer.playerCharacter.domain.model.ChargeRecoveryType`):
  - `NONE`: No charges recovered.
  - `ALL`: Recovers all spent charges up to max (full refill).
  - `FIXED`: Recovers a fixed number of charges specified by `fixedAmount`.
  - `MODIFIER`: Recovers charges equal to the character's ability modifier for the feat's linked ability (minimum 1 if modifier is positive, 0 if ≤ 0).
  - `PROFICIENCY`: Recovers charges equal to the character's proficiency bonus.
  - `LEVEL`: Recovers charges equal to the character's level.

### 1.2 Pure Domain Calculator (`FeatChargeCalculator`)
- Stateless utility class with ZERO framework imports:
  - `calculateMaxCharges(MechanicalEffects effects, CharacterAbilityScores scores, int characterLevel)`:
    - Resolves formula based on `maxChargesType()`.
    - Fallback: if `maxChargesType == null` but legacy `maxUses() != null`, resolves to `maxUses()`.
  - `calculateRecoveryAmount(ChargeRecoveryType recoveryType, Integer fixedAmount, String ability, CharacterAbilityScores scores, int characterLevel)`:
    - Resolves raw recovery quantity according to the declared recovery rule.

### 1.3 Entity Mechanics (`CharacterFeat`)
- `CharacterFeat.recoverCharges(boolean isLongRest, MechanicalEffects effects, CharacterAbilityScores scores, int characterLevel)`:
  - Selects short or long rest recovery rules from `MechanicalEffects`.
  - Fallback: for long rest, defaults to `ALL` if unspecified; for short rest, defaults to `NONE`.
  - Adds computed recovery to `currentCharges`, clamping strictly between `0` and `maxCharges`.
- `CharacterFeat.withMaxCharges(int newMax)`:
  - Returns updated immutable instance with adjusted max charges and clamped `currentCharges`.
