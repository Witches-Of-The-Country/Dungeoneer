# Delta Spec: `player-character-data-model`

> **Change ID:** `002-fix-legacy-items-and-feats-mechanics`
> **Target Canonical Spec:** `openspec/specs/player-character-data-model.md`
> **Change Type:** Modify

## Architectural Updates

### Flyway Migration `V29__fix_legacy_items_and_feats_mechanics.sql`
- Repairs `player_character.items.mechanical_properties` for `half-plate-armor-2014/2024`, `padded-armor-2014/2024`, `hide-armor-2014/2024`, `scale-mail-2014/2024`, `ring-mail-2014/2024`, and `splint-armor-2014/2024`.
- Populates `WondrousMechanics` JSONB on SRD magic items that grant ability bonuses, ability score overrides, or passive AC bonuses.
- Normalizes `player_character.feats.mechanical_effects` JSONB keys (`maxUses` → `fixedMaxCharges`/`usesPerRest`, `rechargeOn` → `restType`, `shortRestRecoveryType`, `longRestRecoveryType`) and backfills `max_charges` and `current_charges` in `player_character.character_feats` for existing character records.
