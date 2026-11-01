# Delta Spec: `player-character-domain`

> **Change ID:** `add-homebrew-creation`
> **Target Canonical Spec:** `openspec/specs/player-character-domain.md`
> **Change Type:** Modify

## Architectural Updates

### 1. Domain Model Changes

- **Ruleset Enum:**
  `com.dungeoneer.playerCharacter.domain.model.Ruleset` is extended with the `HOMEBREW` literal:
  ```java
  public enum Ruleset {
      SRD_2014,
      SRD_2024,
      HOMEBREW
  }
  ```
  The domain layer maintains zero framework dependencies (pure Java).

### 2. Inbound Port: `CreateHomebrewUseCase`

Located in `application.port.in.CreateHomebrewUseCase`:
```java
public interface CreateHomebrewUseCase {
    Spell createSpell(UUID ownerId, CreateHomebrewSpellRequest request);
    Item createItem(UUID ownerId, CreateHomebrewItemRequest request);
    Feat createFeat(UUID ownerId, CreateHomebrewFeatRequest request);
    Lineage createLineage(UUID ownerId, CreateHomebrewLineageRequest request);
    Archetype createArchetype(UUID ownerId, CreateHomebrewArchetypeRequest request);
}
```

### 3. Application DTOs

Located in `application.dto.*` with Jakarta Validation annotations:
- `CreateHomebrewSpellRequest`: `name` (@NotBlank), `level` (@Min(0) @Max(9)), `school` (@NotNull), `castingTime`, `range`, `components`, `duration`, `description` (@NotBlank, @Size(max = 4000)).
- `CreateHomebrewItemRequest`: `name` (@NotBlank), `type` (@NotNull), `cost` (@Min(0)), `weight` (@Min(0)), `description` (@NotBlank, @Size(max = 4000)).
- `CreateHomebrewFeatRequest`: `name` (@NotBlank), `prerequisite`, `description` (@NotBlank, @Size(max = 4000)).
- `CreateHomebrewLineageRequest`: `name` (@NotBlank), `speed` (@Min(0)), `size` (@NotNull), `description` (@NotBlank).
- `CreateHomebrewArchetypeRequest`: `name` (@NotBlank), `parentClass` (@NotBlank), `description` (@NotBlank).

### 4. Outbound Repository Port Enhancements

Repository port interfaces in `application.port.out` are extended with query methods accepting an optional owner ID:
- `List<Spell> findAllVisible(Ruleset ruleset, UUID ownerId)`
- `Spell saveHomebrew(Spell spell, UUID ownerId)`
(And equivalent methods for Item, Feat, Lineage, Archetype).

### 5. Web Controller Endpoints

In `adapter.in.web.controller.ReferenceDataController`:
- `POST /api/reference/spells` (returns HTTP 201 Created with Location header)
- `POST /api/reference/items` (returns HTTP 201 Created)
- `POST /api/reference/feats` (returns HTTP 201 Created)
- `POST /api/reference/lineages` (returns HTTP 201 Created)
- `POST /api/reference/archetypes` (returns HTTP 201 Created)

Endpoints read `X-User-Id` header supplied by the API Gateway to identify the creator.
