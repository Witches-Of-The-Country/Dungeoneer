# player-character-domain Delta for Spec 007

## Summary of Changes
- Update `ReferenceDataRepository`, `ReferenceDataService`, and `ReferenceDataController` to accept multi-select filter collections (levels, schools, categories, rarities, rulesets) and evaluate them using OR logic.
- Ensure `ReferenceDataRepository.findProgressionFeats` and `findSubclassProgressionFeats` correctly return all progression feats up to level 20.
