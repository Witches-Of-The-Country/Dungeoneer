# Delta Spec: `player-character-data-model`

> **Change ID:** `004-fix-character-creation-and-reroll-rules`
> **Target Canonical Spec:** `openspec/specs/player-character-data-model.md`
> **Change Type:** Modify

## Architectural Updates

### Flyway Migration `V32__align_origin_feats_and_reroll_permissions.sql`
- Adds `allow_ability_reroll BOOLEAN NOT NULL DEFAULT FALSE` to `player_character.characters`.
- Aligns `player_character.backgrounds.origin_feat_name` with `player_character.feats.name` and normalizes `feat_type = 'ORIGIN'` across SRD 2024 feats.
