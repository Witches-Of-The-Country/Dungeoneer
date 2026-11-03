Focus: Backend / Web Adapter & Integration Testing
Task: Execute Phase 3: REST Controller, DTOs & Integration Testing

## Execution Context
- Active Plan: `openspec/changes/add-long-short-rest/tasks.md`
- Linked Proposal: `openspec/changes/add-long-short-rest/proposal.md`
- Scope: Execute Phase 3 only. Do not proceed to subsequent phases.

## Implementation Steps
1. Update inbound port and command DTOs in `player-character-service/backend/src/main/java/com/dungeoneer/playerCharacter/application/`:
   - Update `RestUseCase` to support short and long rest execution.
   - Refactor `RestCommand` in `application/dto/`: include rest type (`SHORT`, `LONG`), list of Hit Dice spends (`classProgressionId`, `count`) with Jakarta validation annotations (`@NotNull`, `@Min(1)`).
2. Update character sheet response DTOs:
   - In `CharacterSheetResult`, include `exhaustion` and Hit Dice summary.
   - In `CharacterClassSummaryDto`, include `hitDie`, `hitDiceSpent`, and `availableHitDice`.
   - Update DTO mappers accordingly to populate these values from domain entities.
3. Update `PlayerCharacterController` in `adapter/in/web/controller/`:
   - Endpoint `POST /api/v1/characters/{id}/rest` taking `@Valid @RequestBody RestCommand`.
   - Return HTTP 200 with updated `CharacterSheetResult`.
   - Ensure RFC 7807 `ProblemDetail` is returned for domain and validation errors:
     - Overspending Hit Dice -> HTTP 400.
     - Unowned or foreign `classProgressionId` -> HTTP 400 or HTTP 404.
     - Resting while dead (0 HP) -> HTTP 409 or HTTP 422.
4. Implement Web/MockMvc tests in `player-character-service/backend/src/test/java/com/dungeoneer/playerCharacter/adapter/in/web/controller/`:
   - Test successful rest execution returning updated sheet (TP-03).
   - Test error scenarios: overspending Hit Dice (TE-02), dead character rest attempt (TE-03), invalid/foreign class progression ID (TE-04).
5. Implement or update Testcontainers integration test in `player-character-service/backend/src/test/java/com/dungeoneer/playerCharacter/`:
   - Verify persistence of Flyway V27 columns (`characters.exhaustion` and `class_progressions.hit_dice_spent`) across full rest cycles against a real PostgreSQL container (TP-04).

## Quality Gates & Verification
- Run local validation: `mvn clean test -pl player-character-service/backend`
- Commit locally: `git commit -m "feat(api): expose rest endpoints with DTO validation and integration tests"`

## Phase Handoff & Cumulative Walkthrough
Update the shared walkthrough artifact (`openspec/changes/add-long-short-rest/walkthrough.md` in pt-BR) recording execution progress:
1. List of files created/modified in this phase.
2. Verification and test results.
3. Handoff context prepared for Phase 4.
