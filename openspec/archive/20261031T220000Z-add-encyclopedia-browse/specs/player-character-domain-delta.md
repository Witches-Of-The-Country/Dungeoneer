# Delta Spec: `player-character-domain`

> **Change ID:** `add-encyclopedia-browse`
> **Target Canonical Spec:** `openspec/specs/player-character-domain.md`
> **Change Type:** Modify

## Architectural Updates

### 1. Pure Domain Entity: `Item.java`
- Located in `domain/model/Item.java`.
- Zero framework dependencies (no Spring, no JPA, no Lombok).
- Properties:
  - `id: UUID`
  - `ruleset: Ruleset`
  - `slug: String`
  - `nameEn: String`
  - `name: String`
  - `itemCategory: String`
  - `itemType: String`
  - `rarity: String`
  - `attunement: boolean`
  - `cost: String`
  - `weight: String`
  - `description: String`
- Invariant validations: `name`, `slug`, and `itemType` must not be blank; `ruleset` must not be null.

### 2. Cursor Pagination Domain Contract
- DTO Record: `CursorPageResponse<T>` containing:
  - `items: List<T>`
  - `nextCursor: String` (encoded `name:id` cursor or base64 token)
  - `hasNext: boolean`
  - `pageSize: int`
- Outbound port `ReferenceDataRepository` signature additions:
  - `findSpells(String afterCursor, int limit, String query, Ruleset ruleset, Integer level, String school)`
  - `findFeats(String afterCursor, int limit, String query, Ruleset ruleset)`
  - `findItems(String afterCursor, int limit, String query, Ruleset ruleset, String category, String rarity)`
  - `findArchetypes(String afterCursor, int limit, String query, Ruleset ruleset)`
  - `findLineages(String afterCursor, int limit, String query, Ruleset ruleset)`
