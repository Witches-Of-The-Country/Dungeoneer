# Delta Spec: `player-character-data-model`

> **Change ID:** `add-homebrew-feat-charges`
> **Target Canonical Spec:** `openspec/specs/player-character-data-model.md`
> **Change Type:** Modify

## Architectural Updates
- `feats.mechanical_effects` JSONB gains optional fields: `maxChargesType`, `chargeAbility`, `fixedMaxCharges`, `shortRestRecoveryType`, `shortRestRecoveryFixed`, `longRestRecoveryType`, `longRestRecoveryFixed`. No migration.
