# Delta Spec: `player-character-data-model`

> **Change ID:** `fix-feat-charges-wizard-reference-endpoints`
> **Target Canonical Spec:** `openspec/specs/player-character-data-model.md`
> **Change Type:** Modify

## Architectural Updates
- Migração Flyway atualizando `mechanical_effects` dos feats legados com metadados de recuperação.
- Backfill de `current_charges`/`max_charges` nulos em `player_character.character_feats`.
