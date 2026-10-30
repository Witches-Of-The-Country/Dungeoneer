# Delta Spec: `player-character-data-model`

> **Change ID:** `reference-data-backend`
> **Target Canonical Spec:** `openspec/specs/player-character-data-model.md`
> **Change Type:** Modify

## Architectural Updates

1. **Schema Isolation**: All tables (`characters`, `ability_scores`, `archetypes`, `lineages`, `backgrounds`) reside strictly inside the `player_character` PostgreSQL schema.
2. **`characters` Table Updates**:
   - `user_id`: `UUID NULLABLE` logical reference to `auth-service` user ID (no cross-schema SQL foreign key constraint).
   - `lineage_id`: `UUID NOT NULL` with foreign key `fk_characters_lineage` referencing `player_character.lineages(id)`.
   - `background_id`: `UUID NOT NULL` with foreign key `fk_characters_background` referencing `player_character.backgrounds(id)`.
3. **`archetypes` Table Updates**:
   - Unique constraint: `UNIQUE (name, ruleset, source)` so `SRD_2014` and `SRD_2024` entries can share class names (e.g., `Fighter`).
4. **`lineages` Table Enhancements**:
   - `ability_bonuses JSONB NOT NULL DEFAULT '{}'`: stores `SRD_2014` fixed ASI bonuses (e.g., `{"STR": 2, "CON": 1}` or `{"ALL": 1}` for Human). In `SRD_2024`, remains `{}`.
   - `traits_summary TEXT[] NULL`: array of summary traits (e.g. `Darkvision`, `Dwarven Resilience`) for creation wizard display.
   - Unique constraint: `UNIQUE (name, ruleset, source)`.
5. **`backgrounds` Table Enhancements**:
   - `allowed_ability_bonuses VARCHAR[] NOT NULL DEFAULT '{}'`: stores the 3 allowed abilities for `SRD_2024` ASI distribution (`+2/+1` or `+1/+1/+1`).
   - `origin_feat_name VARCHAR(100) NULL`: informational Origin Feat name for `SRD_2024` (e.g. `Alert`, `Savage Attacker`).
   - `origin_feat_description TEXT NULL`: informational Origin Feat effect summary.
   - Unique constraint: `UNIQUE (name, ruleset, source)`.
