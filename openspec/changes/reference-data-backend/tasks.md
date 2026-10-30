# Tasks: Reference Data Backend

> **Change ID:** `reference-data-backend`
> **Linked Proposal:** [proposal.md](./proposal.md)
> **Status:** in-progress
>
> All tasks must be `[x]` before this change can be archived via `sdd-spec-archive`.

---

## Implementation Tasks

- [x] Write Flyway migration `V2__align_schema_and_create_reference_tables.sql` (create `player_character` schema, move `characters` & `ability_scores`, create `archetypes`, `lineages`, `backgrounds`)
- [x] Write Flyway migration `V3__seed_srd_2014_and_2024_reference_data.sql` (seed 13 classes, 9 lineages, and 6 backgrounds for `SRD_2014` and `SRD_2024`)
- [x] Update `PlayerCharacterJpaEntity` and `AbilityScoresJpaEntity` with `schema = "player_character"`
- [x] Implement pure Java domain models `Archetype`, `Lineage`, `Background`, `HitDiceEnum`, and `SpellcastingType`
- [ ] Create `ReferenceDataRepository` outbound port and `GetReferenceDataUseCase` inbound port
- [ ] Implement `ArchetypeJpaEntity`, `LineageJpaEntity`, `BackgroundJpaEntity`, and `ReferenceDataPersistenceAdapter`
- [ ] Implement `ReferenceDataService` and `ReferenceDataController` (`GET /api/reference/classes`, `/lineages`, `/backgrounds`)

## Security Validation Tasks

- [ ] Validate `ruleset` query parameter against `Ruleset` enum and map invalid values to `422 Unprocessable Entity` ProblemDetail
- [ ] Confirm no secrets or credentials are hardcoded in migrations or configuration

## Shift-Left Test Suite Tasks

### Happy Path
- [x] Write unit tests for `Archetype`, `Lineage`, `Background` domain models (TP-01)
- [ ] Write Testcontainers integration tests for `ReferenceDataPersistenceAdapter` verifying `V2` and `V3` seed counts (13 classes, 9 lineages, 6 backgrounds per ruleset) (TP-02)
- [ ] Write MockMvc tests for `GET /api/reference/classes`, `/lineages`, and `/backgrounds` (TP-03)

### Edge Cases & Failure Modes
- [ ] Write MockMvc test for invalid `?ruleset=INVALID` returning `422` ProblemDetail (TE-01)
- [x] Write domain unit tests for invalid/null constructor arguments (TE-02)

## Documentation Tasks

- [ ] Update `specs/player-character-data-model-delta.md` with reference table ASI and origin feat columns

## Review Gate

- [ ] All implementation tasks complete
- [ ] All security validation tasks complete
- [ ] All tests passing (`mvn -f player-character-service/backend/pom.xml clean verify`)
- [ ] Proposal acceptance criteria verified (run `sdd-spec-validate`)
- [ ] Ready for `sdd-spec-archive`
