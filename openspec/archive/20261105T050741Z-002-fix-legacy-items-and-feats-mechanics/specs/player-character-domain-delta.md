# Delta Spec: `player-character-domain`

> **Change ID:** `002-fix-legacy-items-and-feats-mechanics`
> **Target Canonical Spec:** `openspec/specs/player-character-domain.md`
> **Change Type:** Modify

## Architectural Updates

### Wondrous Item Mechanics & Effective Ability Scores
- `ItemMechanics` sealed hierarchy includes `WondrousMechanics(Map<Ability, Integer> abilityBonuses, Map<Ability, Integer> abilityOverrides, int acBonus, int savingThrowBonus)`.
- `CharacterSheetHydrator` computes effective ability scores for equipped items (requiring `attuned == true` when `requiresAttunement == true`):
  1. Add `abilityBonuses` to base ability scores (capped at 30).
  2. Apply `abilityOverrides` via `Math.max(scoreWithBonuses, overrideValue)` per D&D 5e rules.
  3. Propagate effective ability scores to ability modifiers, saving throws, skill modifiers, weapon attack/damage rolls, AC calculation, and feat `ABILITY_MODIFIER` max charges.

### Rest Charge Recovery Persistence
- `RestService.recoverFeats` returns the updated `List<CharacterFeat>` and updates the aggregate root via `characterToSave = characterToSave.withFeats(recoveredFeats)` prior to `characterRepository.save(characterToSave)`, preventing JPA cascade overwrites.
