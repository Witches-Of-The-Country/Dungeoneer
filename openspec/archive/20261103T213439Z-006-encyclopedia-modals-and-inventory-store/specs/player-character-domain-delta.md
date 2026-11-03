# player-character-domain Delta for Spec 006

## Summary of Changes
- Fix `PlayerCharacter` domain model immutability methods (`withItems`, `withFeats`, `withLevelUp`) to preserve `currencyPouch` and `avatarUrl`.
- Fix `CharacterQueryService` and `UpdateCharacterService` to use `CharacterSheetHydrator.hydrate` instead of static `toSheetResult` with empty maps.
- Expose `source` in `CharacterFeatDto`.
- Add `PUT /api/reference/{category}/{id}` REST endpoints for updating Homebrew reference data.
- Add update methods to `CreateHomebrewUseCase` and `HomebrewService`.

## Endpoints
```http
PUT /api/reference/spells/{id}
PUT /api/reference/feats/{id}
PUT /api/reference/items/{id}
PUT /api/reference/classes/{id}
PUT /api/reference/lineages/{id}
PUT /api/reference/subclasses/{id}
```
Only entities with `ruleset == HOMEBREW` can be modified. Attempting to update SRD entities returns `403 Forbidden` or `400 Bad Request`.
