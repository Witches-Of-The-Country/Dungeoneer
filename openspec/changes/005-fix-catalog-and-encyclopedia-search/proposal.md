# Bugfix Proposal: Fix Text Search in Item Catalog & All Encyclopedia Compendiums

> **Change ID:** `005-fix-catalog-and-encyclopedia-search`
> **Execution Order:** `005`
> **Type:** Bugfix & Legacy Adaptation
> **Priority:** Medium
> **Status:** draft
> **Created:** 2026-11-04
> **Last Updated:** 2026-11-04

---

## Summary

Fixes broken interactive text search across the Item Catalog modal (`item-catalog-modal.tsx`) and all 5 Encyclopedia compendiums (`Items`, `Spells`, `Feats`, `Classes/Archetypes`, `Lineages`). Replaces rigid whole-word PostgreSQL `plainto_tsquery` matching with accent-insensitive, case-insensitive substring and prefix matching (`ILIKE` / `translate` / `unaccent`), and removes the destructive double-filtering loop in the frontend Item Catalog modal.

## Grouped Bugs & Symptoms

- **Bug 1:** Pesquisa por texto quebrada (catálogo de itens e enciclopédia).

## Root Cause Analysis (Causa Raiz)

1. **No Prefix/Substring Matching in `plainto_tsquery`**:
   In `SpringDataItemRepository.java` (line 47), text search uses:
   ```sql
   to_tsvector('portuguese', coalesce(i.name, '') || ' ' || coalesce(i.name_en, '') || ' ' || coalesce(i.description, ''))
   @@ plainto_tsquery('portuguese', CAST(:q AS VARCHAR))
   ```
   `plainto_tsquery` parses tokens into stemmed full words without prefix wildcards (`:*`). Typing partial prefixes like `"esp"` (for `"Espada"`), `"arm"` (for `"Armadura"`), or `"poc"` (for `"Poção"`) generates lexeme `'esp'` which does NOT match the stemmed lexeme `'espad'`, returning `0` rows immediately as the user types!
2. **Accent Sensitivity in Portuguese Queries**:
   Searching `"pocao"`, `"lanca"`, or `"maca"` without diacritics fails to match `"Poção"`, `"Lança"`, or `"Maça"` because PostgreSQL's default `'portuguese'` dictionary does not strip accents.
3. **Destructive Double-Filtering in `item-catalog-modal.tsx`**:
   In `dungeoneer-frontend/components/character-sheet/item-catalog-modal.tsx` (lines 112–170), every keystroke triggers a backend `fetchItems({ q: search.trim() })`. Because the backend returns `[]` on partial words, `setItems([])` wipes the local array, and the subsequent client-side `items.filter(...)` filters an empty array.

## Systemic Ripple Effects & Similar Latent Bugs (Repercussão no Código)

The exact same `plainto_tsquery` flaw exists in **all 5 reference data repositories**:
1. `SpringDataItemRepository.java` (Items)
2. `SpringDataSpellRepository.java` (Spells)
3. `SpringDataFeatRepository.java` (Feats)
4. `SpringDataArchetypeRepository.java` (Classes / Archetypes)
5. `SpringDataLineageRepository.java` (Lineages)

Fixing all 5 repositories ensures uniform, responsive substring/prefix search across the entire Encyclopedia and Character Sheet modals.

## Legacy Code & Data Adaptation Strategy (Adequação de Legado)

### Legacy Impact Matrix
| Legacy Artifact / Table | Discrepancy Identified | Adaptation / Migration Action |
|-------------------------|------------------------|-------------------------------|
| `items`, `spells`, `feats`, `archetypes`, `lineages` | Accented Portuguese names (`Poção`, `Bárbaro`, `Bênção`) vs unaccented user queries (`pocao`, `barbaro`, `bencao`) | Use PostgreSQL `translate(lower(...), 'áàâãäéèêëíìîïóòôõöúùûüçñ', 'aaaaaeeeeiiiiooooouuuucn')` with `LIKE '%' || translate(lower(:q), ...) || '%'` across `name`, `name_en`, and `description` so no PostgreSQL superuser extension is required and all legacy rows match seamlessly. |

## Scope

### In Scope
- Update `SpringDataItemRepository`, `SpringDataSpellRepository`, `SpringDataFeatRepository`, `SpringDataArchetypeRepository`, and `SpringDataLineageRepository` to use accent-insensitive, case-insensitive substring matching (`LIKE` with `translate(lower(...))`) alongside or replacing strict `plainto_tsquery`.
- Include `item_type` and `item_category` in `SpringDataItemRepository` text matching.
- Fix `dungeoneer-frontend/components/character-sheet/item-catalog-modal.tsx` to debounce search cleanly and support accent-insensitive filtering.

### Out of Scope
- External Elasticsearch / Algolia cluster deployment.

## Proposed Solution

### Impacted Files
| File | Change Type | Notes |
|------|-------------|-------|
| `player-character-service/.../adapter/out/persistence/SpringDataItemRepository.java` | Modify | Accent-insensitive & case-insensitive substring search (`ILIKE` + `translate`) |
| `player-character-service/.../adapter/out/persistence/SpringDataSpellRepository.java` | Modify | Accent-insensitive & case-insensitive substring search |
| `player-character-service/.../adapter/out/persistence/SpringDataFeatRepository.java` | Modify | Accent-insensitive & case-insensitive substring search |
| `player-character-service/.../adapter/out/persistence/SpringDataArchetypeRepository.java` | Modify | Accent-insensitive & case-insensitive substring search |
| `player-character-service/.../adapter/out/persistence/SpringDataLineageRepository.java` | Modify | Accent-insensitive & case-insensitive substring search |
| `dungeoneer-frontend/components/character-sheet/item-catalog-modal.tsx` | Modify | Normalize accent-insensitive client filter and debounce backend search |

## Acceptance Criteria

- [ ] **Given** a user in the Item Catalog or Encyclopedia **When** they type a partial word such as `"esp"` or `"arm"` **Then** matching items (`"Espada Longa"`, `"Armadura de Placas"`, etc.) are returned immediately.
- [ ] **Given** a user searching without accents such as `"pocao"` or `"bencao"` **When** the search executes **Then** accented entries (`"Poção de Cura"`, `"Bênção"`) are returned.
- [ ] **Given** a user searching by English name (`"Longsword"`, `"Fireball"`) **When** the search executes **Then** the corresponding items/spells are returned across all 5 Encyclopedia categories.

## Defensive Security & Validation Considerations

### Input Validation
- Search parameter `:q` Remains strictly bound as a prepared statement parameter (`@Param("q")`), preventing SQL injection.

### Secret Isolation
- N/A.

### Authorization & Authentication Impact
- Public/authenticated reference endpoints retain existing visibility rules for `SRD_2014`, `SRD_2024`, and `HOMEBREW`.

### Other Security Risks
- Bound query length and pagination (`Pageable`) prevent ReDoS or unbounded result sets.

## Testing Strategy & Shift-Left Plan

### Regression & Happy Path Tests
| Test ID | Scenario | Test Type | Expected Result |
|---------|----------|-----------|-----------------|
| TR-01 | Search items with partial prefix `q = "esp"` | Integration / Repository | Returns `"Espada Longa"`, `"Espada Curta"`, etc. |
| TR-02 | Search items without accents `q = "pocao"` | Integration / Repository | Returns `"Poção de Cura"` |
| TR-03 | Type partial query in `ItemCatalogModal` | Frontend Unit (Vitest) | Filtered list displays matching items without wiping to empty |

### Edge Case & Failure Mode Tests
| Test ID | Scenario | Test Type | Expected Result |
|---------|----------|-----------|-----------------|
| TE-01 | Search with `q = ""` or `q = "   "` | Unit / Integration | Returns full paginated list without error |

### Test Commands
- **Backend tests:** `mvn clean test`
- **Frontend tests:** `npm test`

## Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| `unaccent` extension not installed in all PostgreSQL environments | Med | High | Use standard built-in SQL `translate(lower(...), 'áàâãäéèêëíìîïóòôõöúùûüçñ', 'aaaaaeeeeiiiiooooouuuucn')` which works on 100% of PostgreSQL instances without extensions |

## Specification Changes & Delta Stubs

### `specs/player-character-domain-delta.md`
- **Target Canonical Spec:** `openspec/specs/player-character-domain.md`
- **Summary of Updates:** Document accent-insensitive substring search behavior across all `/api/reference/*` endpoints.

### `specs/frontend-web-app-delta.md`
- **Target Canonical Spec:** `openspec/specs/frontend-web-app.md`
- **Summary of Updates:** Document debounced, accent-tolerant search in `ItemCatalogModal` and `Encyclopedia`.
