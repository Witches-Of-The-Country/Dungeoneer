# Delta Spec: `player-character-data-model`

> **Change ID:** `reference-data-backend`
> **Target Canonical Spec:** `openspec/specs/player-character-data-model.md`
> **Change Type:** Modify

## Architectural Updates

1. **Schema Isolation**: All tables (`characters`, `ability_scores`, `archetypes`, `lineages`, `backgrounds`) reside strictly inside the `player_character` PostgreSQL schema.
2. **`lineages` Table Enhancements**:
   - Add `ability_bonuses JSONB NOT NULL DEFAULT '{}'` to store `SRD_2014` fixed ASI bonuses (e.g., `{"STR": 2, "CON": 1}`).
   - Add `traits_summary TEXT[]` for displaying racial/lineage traits in the creation wizard.
3. **`backgrounds` Table Enhancements**:
   - Add `allowed_ability_bonuses VARCHAR[]` to store the 3 allowed abilities for `SRD_2024` ASI distribution (`+2/+1` or `+1/+1/+1`).
   - Add `origin_feat_name VARCHAR(100) NULL` and `origin_feat_description TEXT NULL` for informational display of `SRD_2024` Origin Feats.
4. **Unique Constraint on Reference Tables**:
   - `UNIQUE (name, ruleset, source)` on `archetypes`, `lineages`, and `backgrounds` so `SRD_2014` and `SRD_2024` entries coexist cleanly.
