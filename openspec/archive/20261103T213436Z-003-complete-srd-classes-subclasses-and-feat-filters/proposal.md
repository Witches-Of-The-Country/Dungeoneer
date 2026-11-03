# Change Proposal: 003 — Complete SRD 2014 & 2024 Class Feats, Subclasses & Encyclopedia Multi-Select Feat Filter

> **Change ID:** `003-complete-srd-classes-subclasses-and-feat-filters`
> **Status:** approved
> **Created:** 2026-11-01

---

## Summary

1. Seed the complete catalog of Level 1–20 Class Features (`archetype_feats`), additional official Subclasses (`subclasses`), and Subclass Progression Features (`subclass_feats`) across all 13 classes in both `SRD_2014` and `SRD_2024`.
2. Add a Multi-Select filter by Feat Type (`ORIGIN`, `GENERAL`, `LINEAGE`, `CLASS`, `SUBCLASS`) on the `/encyclopedia/feats` page and backend `GET /api/reference/feats` endpoint.

---

## Motivation & Problem Statement

- Currently, `V17__seed_srd_subclasses_and_progression_feats.sql` only seeded 1 subclass per class, left 8 classes (`Cleric`, `Druid`, `Monk`, `Ranger`, `Sorcerer`, `Warlock`, `Wizard`, `Artificer`) with zero class progression feats, and left 7 subclasses with zero subclass feats. Players leveling up or browsing `/encyclopedia/classes` need the complete progression features and multiple subclasses per class.
- On `/encyclopedia/feats`, all feat categories (`ORIGIN`, `GENERAL`, `CLASS`, `SUBCLASS`, `LINEAGE`) are mixed together without a multi-select filter by `featType`.

---

## Scope

### In Scope
- Creating Flyway migration `V20__seed_complete_srd_classes_and_subclasses_progression.sql` seeding Level 1–20 class features (`archetype_feats`) for all 13 classes and multiple subclasses + `subclass_feats` per class across `SRD_2014` and `SRD_2024`.
- Updating `GET /api/reference/feats` (`ReferenceDataController`, `GetReferenceDataUseCase`, `ReferenceDataService`, `ReferenceDataPersistenceAdapter`, `SpringDataFeatRepository`) to accept an optional `List<String> featTypes` filter.
- Adding an interactive multi-select `featType` filter bar (`Origem`, `Geral`, `Classe`, `Subclasse`, `Linhagem`) on `/encyclopedia/feats`.

### Out of Scope
- Modifying Homebrew feat creation forms (covered in `002-advanced-homebrew-mechanics-and-prerequisites`).

---

## Proposed Solution

1. **Flyway Migration (`V20__seed_complete_srd_classes_and_subclasses_progression.sql`)**:
   - Seed Level 1–20 class features (`archetype_feats`) for all 13 classes (`Barbarian`, `Bard`, `Cleric`, `Druid`, `Fighter`, `Monk`, `Paladin`, `Ranger`, `Rogue`, `Sorcerer`, `Warlock`, `Wizard`, `Artificer`) across `SRD_2014` and `SRD_2024` using idempotent `ON CONFLICT DO NOTHING` inserts.
   - Seed additional iconic subclasses per class (e.g., Fighter: *Battle Master*, *Eldritch Knight*; Barbarian: *Path of the Totem Warrior / Wild Heart*; Rogue: *Assassin*, *Arcane Trickster*; Cleric: *Light Domain*, *War Domain*; Wizard: *School of Abjuration*, *School of Divination*; Warlock: *The Archfey*, *The Great Old One*; Sorcerer: *Wild Magic*; Paladin: *Oath of Vengeance*, *Oath of the Ancients*; Bard: *College of Valor*; Druid: *Circle of the Moon*; Monk: *Way of Shadow*, *Way of the Four Elements*; Ranger: *Beast Master*, *Gloom Stalker*; Artificer: *Armorer*, *Battle Smith*) and populate all missing `subclass_feats` across Levels 1–20.
2. **Backend Multi-Select `featTypes` Filter (`ReferenceDataController`, `SpringDataFeatRepository`, `ReferenceDataPersistenceAdapter`)**:
   - Accept optional `@RequestParam(required = false) List<String> featTypes` on `GET /api/reference/feats`.
   - Validate each token against `FeatType` enum and filter `feats` where `feat_type IN (:featTypes)` when non-empty.
3. **Frontend Multi-Select Filter on `/encyclopedia/feats` (`app/encyclopedia/feats/page.tsx`, `lib/api/encyclopedia.ts`)**:
   - Render interactive multi-select filter chips (`Origem`, `Geral`, `Classe`, `Subclasse`, `Linhagem`) on `/encyclopedia/feats` and pass `featTypes` to `useInfiniteFeats`.

---

## Impacted Files

| File Path | Layer | Reason |
|---|---|---|
| `player-character-service/.../db/migration/V20__seed_complete_srd_classes_and_subclasses_progression.sql` | Database | Seed all missing class feats, additional subclasses, and subclass feats. |
| `player-character-service/.../ReferenceDataController.java` | Web | Accept `featTypes` query parameter on `GET /api/reference/feats`. |
| `player-character-service/.../SpringDataFeatRepository.java` | Persistence | Support filtering by `featTypes` list. |
| `dungeoneer-frontend/lib/api/encyclopedia.ts` | Frontend API | Add `featTypes?: string[]` to `FeatFilters` and query string builder. |
| `dungeoneer-frontend/app/encyclopedia/feats/page.tsx` | Frontend Page | Render multi-select `featType` filter bar. |

---

## Acceptance Criteria

- **Given** any of the 13 classes in `/encyclopedia/classes` (`SRD_2014` or `SRD_2024`), **When** the user opens `ClassDetailModal` or levels up a character, **Then** every class displays its Level 1–20 class features and multiple subclasses with their respective subclass progression features.
- **Given** a user on `/encyclopedia/feats`, **When** they select one or more feat type chips (e.g. `Origem` + `Geral`), **Then** `GET /api/reference/feats` returns only feats matching the selected `featType` values.

---

## Defensive Security & Validation Considerations

- **Enum Whitelist Validation**: `ReferenceDataService` validates every element in `featTypes` against the `FeatType` enum (`GENERAL`, `ORIGIN`, `CLASS`, `SUBCLASS`, `LINEAGE`), ignoring or rejecting unknown tokens and capping the filter list size at `10` elements to prevent query parameter abuse.
- **Idempotent SQL Migrations**: All `INSERT` statements in `V20` use deterministic UUIDs/slugs and `ON CONFLICT DO NOTHING` to guarantee safe execution across clean and existing databases.
- **Auth Impact**: `GET /api/reference/feats` remains a public/authenticated read-only reference endpoint with user-scoped Homebrew isolation.

---

## Testing Strategy & Shift-Left Plan

| Scenario | Type | Target Test Class / File | Expected Outcome |
|---|---|---|---|
| Query `GET /api/reference/feats` with `featTypes=ORIGIN,GENERAL` | Unit | `ReferenceDataServiceTest` / `ReferenceDataPersistenceAdapterTest` | Returns only feats whose `featType` is in `['ORIGIN', 'GENERAL']`. |
| Query `GET /api/reference/feats` with empty `featTypes` or invalid token | Unit | `ReferenceDataServiceTest` | Empty `featTypes` returns all feat types; invalid token is safely sanitized/rejected. |
| Multi-select chip interaction on `/encyclopedia/feats` | Unit | `dungeoneer-frontend/__tests__/encyclopedia-browse.test.tsx` | Clicking `Origem` and `Geral` chips updates filter state and triggers query with `featTypes: ['ORIGIN', 'GENERAL']`. |

- **Verification Commands**:
  - `mvn -f player-character-service/backend/pom.xml test`
  - `npm --prefix dungeoneer-frontend run test:unit`

---

## Risks and Mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| Slug collisions in `V20` migration with existing `V17` rows | Medium | Use `ON CONFLICT (ruleset, slug) DO NOTHING` and `ON CONFLICT DO NOTHING` on join tables (`archetype_feats`, `subclass_feats`). |
| Multi-value query parameter encoding differences (`featTypes=A,B` vs `featTypes=A&featTypes=B`) | Low | Spring `@RequestParam List<String> featTypes` natively supports both comma-separated and repeated query parameters. |
