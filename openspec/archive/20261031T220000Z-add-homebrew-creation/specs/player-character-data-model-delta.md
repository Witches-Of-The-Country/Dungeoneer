# Delta Spec: `player-character-data-model`

> **Change ID:** `add-homebrew-creation`
> **Target Canonical Spec:** `openspec/specs/player-character-data-model.md`
> **Change Type:** Modify

## Architectural Updates

### 1. Flyway Migration `V8__enable_homebrew_ruleset.sql`

Reference tables in schema `player_character` are updated to support user-authored Homebrew content:

- **Altered Tables:** `spell`, `feat`, `item`, `lineage`, `archetype`.
- **Constraint Changes:**
  The `ruleset` CHECK constraint on each table is updated from `('SRD_2014', 'SRD_2024')` to `('SRD_2014', 'SRD_2024', 'HOMEBREW')`.
- **Column Additions:**
  Each reference table receives a nullable `owner_id UUID NULL` column.
  - Value is `NULL` for standard SRD 2014/2024 reference records.
  - Value is populated with the creating user's UUID for records where `ruleset = 'HOMEBREW'`.
- **Indexes:**
  Composite index added on `(ruleset, owner_id)` for each reference table to optimize filtered queries:
  - `idx_spell_owner_ruleset`
  - `idx_feat_owner_ruleset`
  - `idx_item_owner_ruleset`
  - `idx_lineage_owner_ruleset`
  - `idx_archetype_owner_ruleset`

### 2. Visibility Query Contract

Repository persistence queries for reference data must implement the visibility predicate:
```sql
WHERE ruleset IN (:rulesets) 
  AND (owner_id IS NULL OR owner_id = :userId)
```
When `userId` is omitted (unauthenticated public requests), the query defaults to:
```sql
WHERE ruleset IN (:rulesets) 
  AND owner_id IS NULL
```
