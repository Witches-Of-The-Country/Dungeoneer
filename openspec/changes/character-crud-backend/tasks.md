# Tasks: Character Crud Backend

> **Change ID:** `character-crud-backend`
> **Linked Proposal:** [proposal.md](./proposal.md)
> **Status:** in-progress
>
> All tasks must be `[x]` before this change can be archived via `sdd-spec-archive`.

---

## Implementation Tasks

- [x] Create Flyway migration `V4__add_character_crud_and_roll_tables.sql` (`class_progressions`, `ability_rolls` with partial unique index for unused roll per user, `hp_rolls`, and `notes`/`hp_calculation_method`/`has_pending_level_choices`/`user_id NOT NULL` on `characters`)
- [x] Remove duplicate dormant stubs (`CreatePlayerCharacterUseCase`, `CharacterRepositoryPort`, `CreatePlayerCharacterService`, `PlayerCharacterUseCaseConfig`)
- [ ] Implement `AbilityScoreValidator`, `RulesEngine` (`Srd2014RulesEngine`, `Srd2024RulesEngine`), `ClassProgression`, and HP calculation (`MAX`, `AVERAGE`, `ROLLED`) in `domain/model/`
- [ ] Update `PlayerCharacter` domain model with `notes`, `hpCalculationMethod`, `hasPendingLevelChoices`, `ClassProgression`, and `updateSheet(name, hpCurrent, hpTemp, notes)`
- [ ] Implement `RollAbilityScoresUseCase` and `RollHitPointsUseCase` (single-chance idempotent roll generation and consumption)
- [ ] Update `CreateCharacterUseCase` / `CreateCharacterService` to validate ability score methods (`STANDARD_ARRAY`, `POINT_BUY`, `ROLLED`), apply ruleset ASI, calculate HP, and persist `ClassProgression`
- [ ] Implement `GetCharacterUseCase`, `ListCharactersUseCase`, `UpdateCharacterUseCase`, and `DeleteCharacterUseCase` scoped by `userId`
- [ ] Update `PlayerCharacterPersistenceAdapter` and JPA entities for `ClassProgression`, `AbilityRoll`, `HpRoll`, and user-scoped queries
- [ ] Update `PlayerCharacterController` (`POST /ability-rolls`, `POST /hp-rolls`, `POST`, `GET`, `GET/{id}`, `PUT/{id}`, `DELETE/{id}`) with `@RequestHeader("X-User-Id") UUID userId`
- [ ] Update `GlobalExceptionHandler` to map `CharacterNotFoundException` (`404`), `CharacterValidationException` (`422`), `RollConflictException` (`409`), and missing `X-User-Id` (`401`)

## Security Validation Tasks

- [ ] Verify all character and roll queries filter by `userId` (`X-User-Id`) to prevent IDOR / BOLA
- [ ] Verify `PUT /api/characters/{id}` only mutates `name`, `hitPointsCurrent`, `hitPointsTemp`, and `notes`
- [ ] Verify `ROLLED` ability scores and HP rolls cannot be reused once `used = true` or forged by the client

## Shift-Left Test Suite Tasks

### Happy Path
- [ ] Write unit tests for `AbilityScoreValidator`, `Srd2014RulesEngine`, `Srd2024RulesEngine`, and HP calculation methods (TP-01)
- [ ] Write unit and integration tests for single-chance `ability-rolls` and `hp-rolls` idempotency and consumption (TP-02)
- [ ] Write Testcontainers integration and MockMvc API tests for `POST`, `GET`, `GET/{id}`, `PUT/{id}`, and `DELETE/{id}` (TP-03)

### Edge Cases & Failure Modes
- [ ] Write unit and MockMvc tests for tampered `ROLLED` scores, reused `abilityRollId`, and Point Buy > 27 points (TE-01)
- [ ] Write unit and MockMvc tests for cross-user access returning `404 Not Found` (TE-02)
- [ ] Write unit tests for invalid `SRD_2024` background ASI distribution (TE-03)

## Documentation Tasks

- [ ] Update `specs/player-character-domain-delta.md` and `specs/player-character-data-model-delta.md`

## Review Gate

- [ ] All implementation tasks complete
- [ ] All security validation tasks complete
- [ ] All tests passing (`mvn -f player-character-service/backend/pom.xml clean verify`)
- [ ] Proposal acceptance criteria verified (run `sdd-spec-validate`)
- [ ] Ready for `sdd-spec-archive`
