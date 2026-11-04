# player-character-data-model Delta for Spec 007

## Summary of Changes
- Add Flyway migration `V25__seed_missing_srd_class_and_subclass_progression_feats.sql` seeding:
  - Class progression feats for levels 1 to 20 for all 12 SRD classes.
  - Subclass progression feats for official SRD subclasses across tiers.
  - Associations in `player_character.archetype_feats` and `player_character.subclass_feats`.
- In `player_character.items`, ensure category/type query matching allows filtering by specific types (`Poção`, `Item Maravilhoso`, `Equipamento de Aventura`).
