# Delta Spec: `player-character-domain`

> **Change ID:** `005-fix-catalog-and-encyclopedia-search`
> **Target Canonical Spec:** `openspec/specs/player-character-domain.md`
> **Change Type:** Modify

## Architectural Updates

### Reference Data Text Search (`q` parameter)
- All reference catalog queries (`/api/reference/items`, `/spells`, `/feats`, `/classes`, `/lineages`) perform case-insensitive and accent-insensitive substring matching (`translate(lower(...)) LIKE '%' || translate(lower(:q)) || '%'`) across `name`, `name_en`, and `description`, supporting partial prefixes as the user types.
