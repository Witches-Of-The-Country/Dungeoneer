# Delta: Player Character Domain — Half-Feat Ability Choice on Level Up

> **Target Specification:** `openspec/specs/player-character-domain.md`  
> **Change ID:** `level-up-half-feat-ability-choice`

---

## Domain Rules Updates (UC2: Level Up & Multiclass)

### Half-Feat Ability Choice Mechanics
When leveling up a character to an ASI threshold level (e.g. 4, 8, 12, 16, 19, or class-specific levels like Fighter 6/14, Rogue 10) and selecting a Feat:
1. **Feat with `abilityBonusChoice`**:
   - The feat's `MechanicalEffects.abilityBonusChoice` defines `chooseCount` (typically 1), `allowedAbilities` (e.g. `["STR", "DEX"]`), and `bonusAmount` (typically 1).
   - The client MUST provide `abilityScoreIncreases` containing exactly `chooseCount` ability entries.
   - Each selected ability must normalize to one of the feat's `allowedAbilities`.
   - Each bonus value must equal `bonusAmount`.
   - Rejects missing choices, unauthorized abilities, or invalid bonus values with `CharacterValidationException` (HTTP 422 Unprocessable Entity).
2. **Feat with fixed `abilityBonuses`**:
   - Server continues to automatically apply declared fixed bonuses (e.g. `{"DEX": 1}`).
   - Rejects client-supplied `abilityScoreIncreases` that do not match the fixed bonuses.
3. **Feats without ability bonuses**:
   - If `abilityScoreIncreases` is supplied with a non-half feat, server rejects the request with `CharacterValidationException`.
4. **Score Capping**:
   - Attribute increases applied from half-feats remain capped at 20.
